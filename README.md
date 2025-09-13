# habit_tracker_full_with_blocking.py
"""
ZENTRACK — Habit Tracker (full) with:
 - SQLite storage (usage + goals)
 - Manual & Automatic tracking (active-window on Windows when available)
 - Productivity classification using rules (editable)
 - Goals, notifications, export, seed demo
 - Blocking system for unproductive apps (manual + automatic)
 - Settings GUI to edit unproductive-app list and usage-threshold (persists to settings.json)
 - Charts: Today's pie and trends (7/30 day)
"""

import sqlite3
import threading
import time
import datetime
import sys
import os
import json
import random
import psutil  # required for process listing / killing
import traceback

import tkinter as tk
from tkinter import ttk, messagebox, simpledialog, filedialog

import matplotlib
matplotlib.use("TkAgg")
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from matplotlib.figure import Figure

print("🚀 Habit Tracker is starting...")

# optional notifications
try:
    from plyer import notification
except Exception:
    notification = None

# windows active-window helpers
WIN_ACTIVE_OK = False
if sys.platform.startswith("win"):
    try:
        import win32gui
        import win32process
        WIN_ACTIVE_OK = True
    except Exception:
        WIN_ACTIVE_OK = False

# ----------------- Files & defaults -----------------
DB_FILE = "habit_tracker.db"
RULES_FILE = "rules.json"
SETTINGS_FILE = "settings.json"

POLL_INTERVAL_SECONDS = 8  # auto-check interval for active window
NOTIFY_UNPRODUCTIVE_THRESHOLD = 3600  # fallback threshold (seconds) if not using settings

DEFAULT_PRODUCTIVE = ["code", "vscode", "pycharm", "sublime", "terminal", "word", "excel", "zoom", "teams", "slack"]
DEFAULT_UNPRODUCTIVE = ["youtube", "instagram", "facebook", "tiktok", "netflix", "spotify", "whatsapp", "chrome"]

DEFAULT_SETTINGS = {
    "unproductive_apps": DEFAULT_UNPRODUCTIVE.copy(),
    # threshold in minutes for per-app daily usage to trigger automatic blocking
    "usage_limit_minutes": 30,
    # block duration minutes
    "block_duration_minutes": 60
}

# ----------------- Database -----------------
class DB:
    def __init__(self, path=DB_FILE):
        self.conn = sqlite3.connect(path, check_same_thread=False)
        self._create_tables()

    def _create_tables(self):
        c = self.conn.cursor()
        c.execute("""
            CREATE TABLE IF NOT EXISTS usage (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                app_name TEXT,
                start_ts INTEGER,
                end_ts INTEGER,
                duration_seconds INTEGER,
                date TEXT
            )
        """)
        c.execute("""
            CREATE TABLE IF NOT EXISTS goals (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT,
                target_seconds INTEGER,
                period TEXT,
                active INTEGER DEFAULT 1
            )
        """)
        self.conn.commit()

    def add_usage(self, app_name, start_ts, end_ts):
        duration = int(end_ts - start_ts)
        if duration <= 0:
            return
        date = datetime.datetime.fromtimestamp(start_ts).strftime("%Y-%m-%d")
        c = self.conn.cursor()
        c.execute("INSERT INTO usage (app_name, start_ts, end_ts, duration_seconds, date) VALUES (?, ?, ?, ?, ?)",
                  (app_name, int(start_ts), int(end_ts), duration, date))
        self.conn.commit()

    def add_manual_usage(self, app_name, seconds, date=None):
        if date is None:
            date = datetime.date.today().strftime("%Y-%m-%d")
        now_ts = int(time.time())
        start_ts = now_ts - seconds
        self.add_usage(app_name, start_ts, now_ts)

    def query_usage_by_date(self, date):
        c = self.conn.cursor()
        c.execute("SELECT app_name, SUM(duration_seconds) FROM usage WHERE date = ? GROUP BY app_name", (date,))
        return c.fetchall()

    def query_usage_between(self, start_date, end_date):
        c = self.conn.cursor()
        c.execute("SELECT app_name, SUM(duration_seconds) FROM usage WHERE date >= ? AND date <= ? GROUP BY app_name",
                  (start_date, end_date))
        return c.fetchall()

    def query_daily_totals(self, days=30):
        results = []
        today = datetime.date.today()
        for i in range(days - 1, -1, -1):
            d = (today - datetime.timedelta(days=i)).strftime("%Y-%m-%d")
            c = self.conn.cursor()
            c.execute("SELECT app_name, SUM(duration_seconds) FROM usage WHERE date = ? GROUP BY app_name", (d,))
            prod = 0
            unprod = 0
            neutral = 0
            for app, sec in c.fetchall():
                if app is None:
                    continue
                key = app.lower()
                if any(k in key for k in RULES["productive"]):
                    prod += sec
                elif any(k in key for k in RULES["unproductive"]):
                    unprod += sec
                else:
                    neutral += sec
            results.append((d, prod or 0, unprod or 0, neutral or 0))
        return results

    def add_goal(self, name, target_seconds, period='daily'):
        c = self.conn.cursor()
        c.execute("INSERT INTO goals (name, target_seconds, period, active) VALUES (?, ?, ?, 1)",
                  (name, target_seconds, period))
        self.conn.commit()

    def get_goals(self):
        c = self.conn.cursor()
        c.execute("SELECT id, name, target_seconds, period, active FROM goals")
        return c.fetchall()

    def toggle_goal(self, goal_id, active):
        c = self.conn.cursor()
        c.execute("UPDATE goals SET active = ? WHERE id = ?", (1 if active else 0, goal_id))
        self.conn.commit()

    def remove_goal(self, goal_id):
        c = self.conn.cursor()
        c.execute("DELETE FROM goals WHERE id = ?", (goal_id,))
        self.conn.commit()

db = DB()

# ----------------- Rules & Settings -----------------
def load_rules():
    if os.path.exists(RULES_FILE):
        try:
            with open(RULES_FILE, "r", encoding="utf-8") as f:
                j = json.load(f)
                prod = [s.lower() for s in j.get("productive", DEFAULT_PRODUCTIVE)]
                unprod = [s.lower() for s in j.get("unproductive", DEFAULT_UNPRODUCTIVE)]
                return {"productive": prod, "unproductive": unprod}
        except Exception:
            pass
    return {"productive": [s.lower() for s in DEFAULT_PRODUCTIVE], "unproductive": [s.lower() for s in DEFAULT_UNPRODUCTIVE]}

def save_rules_to_file(prod_list, unprod_list):
    try:
        with open(RULES_FILE, "w", encoding="utf-8") as f:
            json.dump({"productive": prod_list, "unproductive": unprod_list}, f, indent=2)
    except Exception:
        pass

RULES = load_rules()

def load_settings():
    if os.path.exists(SETTINGS_FILE):
        try:
            with open(SETTINGS_FILE, "r", encoding="utf-8") as f:
                s = json.load(f)
                # ensure keys
                for k, v in DEFAULT_SETTINGS.items():
                    if k not in s:
                        s[k] = v
                return s
        except Exception:
            pass
    # default
    save_settings(DEFAULT_SETTINGS)
    return DEFAULT_SETTINGS.copy()

def save_settings(s):
    try:
        with open(SETTINGS_FILE, "w", encoding="utf-8") as f:
            json.dump(s, f, indent=2)
    except Exception:
        pass

SETTINGS = load_settings()

# ----------------- Active window detection -----------------
def get_active_window_title():
    try:
        if WIN_ACTIVE_OK:
            hwnd = win32gui.GetForegroundWindow()
            pid = win32process.GetWindowThreadProcessId(hwnd)[1]
            try:
                proc = psutil.Process(pid)
                name = proc.name()
            except Exception:
                name = win32gui.GetWindowText(hwnd) or "unknown"
            return (name or "unknown").lower()
        else:
            return None
    except Exception:
        return None

# ----------------- Helpers -----------------
def seconds_to_hms(sec):
    sec = int(sec)
    h = sec // 3600
    m = (sec % 3600) // 60
    s = sec % 60
    return f"{h}h {m}m {s}s"

def classify_app(app_name):
    key = (app_name or "").lower()
    if any(k in key for k in RULES["productive"]):
        return "productive"
    if any(k in key for k in RULES["unproductive"]):
        return "unproductive"
    return "neutral"

def compute_productivity_for_date(date_str):
    rows = db.query_usage_by_date(date_str)
    totals = {"productive": 0, "unproductive": 0, "neutral": 0}
    for app, sec in rows:
        cls = classify_app(app)
        totals[cls] += sec
    return totals

# ----------------- Background tracker -----------------
class AutoTracker(threading.Thread):
    def __init__(self, poll_interval=POLL_INTERVAL_SECONDS):
        super().__init__(daemon=True)
        self.running = False
        self.poll = poll_interval
        self.current_app = None
        self.current_start = None

    def start_tracking(self):
        if not WIN_ACTIVE_OK:
            return False
        self.running = True
        if not self.is_alive():
            self.start()
        return True

    def stop_tracking(self):
        self.running = False
        self._flush_current()

    def _flush_current(self):
        if self.current_app and self.current_start:
            now_ts = int(time.time())
            db.add_usage(self.current_app, self.current_start, now_ts)
            # after flushing, check if this app needs blocking (auto-block)
            maybe_auto_block(self.current_app)
            self.current_app = None
            self.current_start = None

    def run(self):
        while True:
            if not self.running:
                time.sleep(1)
                continue
            try:
                app = get_active_window_title()
                now = int(time.time())
                if app is None:
                    time.sleep(self.poll)
                    continue
                if self.current_app is None:
                    self.current_app = app
                    self.current_start = now
                elif self.current_app != app:
                    # close previous session
                    db.add_usage(self.current_app, self.current_start, now)
                    maybe_auto_block(self.current_app)
                    self.current_app = app
                    self.current_start = now
            except Exception:
                pass
            time.sleep(self.poll)

tracker = AutoTracker()

# ----------------- Blocking system -----------------
# blocked_apps: dict mapping normalized process name -> unblock_timestamp
blocked_apps = {}
blocked_apps_lock = threading.Lock()

def is_blocked(proc_name):
    with blocked_apps_lock:
        t = blocked_apps.get(proc_name)
        if t and time.time() < t:
            return True
        elif t and time.time() >= t:
            # expired
            del blocked_apps[proc_name]
            return False
        return False

def block_app(proc_name, minutes=60):
    with blocked_apps_lock:
        unblock_time = time.time() + minutes * 60
        blocked_apps[proc_name] = unblock_time
    # notify user
    try:
        send_notification("App blocked", f"{proc_name} blocked for {minutes} minutes.")
    except Exception:
        pass
    return unblock_time

def unblock_app(proc_name):
    with blocked_apps_lock:
        if proc_name in blocked_apps:
            del blocked_apps[proc_name]

def monitor_blocked_apps_thread():
    """Background thread: if any blocked app is running, attempt to kill it."""
    while True:
        try:
            with blocked_apps_lock:
                current_blocked = list(blocked_apps.keys())
            if current_blocked:
                for proc in psutil.process_iter(['pid', 'name']):
                    try:
                        name = (proc.info.get('name') or "").lower()
                        # some apps may appear as "chrome.exe" but our blocked key could be "chrome" or vice versa.
                        for blocked_key in current_blocked:
                            if blocked_key and blocked_key in name:
                                try:
                                    proc.kill()
                                except Exception:
                                    pass
                    except (psutil.NoSuchProcess, psutil.AccessDenied):
                        continue
        except Exception:
            # log and continue
            traceback.print_exc()
        time.sleep(5)

def maybe_auto_block(app_process_name):
    """Check daily totals for this app; if threshold reached and app is unproductive, block it automatically."""
    # app_process_name might be window title or process name; we match using RULES["unproductive"] elements
    try:
        today = datetime.date.today().strftime("%Y-%m-%d")
        rows = db.query_usage_by_date(today)
        # find matching row for this app
        total_for_app = 0
        app_key = (app_process_name or "").lower()
        for app, sec in rows:
            if app is None:
                continue
            if app_key in (app or "").lower() or any(k in (app or "").lower() for k in RULES["unproductive"]):
                total_for_app += sec
        # threshold in seconds
        thr = int(SETTINGS.get("usage_limit_minutes", 30)) * 60
        if total_for_app >= thr:
            # choose a process name key to block — try to pick a direct match from the unproductive list
            # prefer exact item from settings["unproductive_apps"] that matches a substring
            candidate = None
            for item in SETTINGS.get("unproductive_apps", []):
                if item.lower() in app_key or app_key in item.lower():
                    candidate = item.lower()
                    break
            # fallback to RULES list element
            if not candidate:
                for item in RULES.get("unproductive", []):
                    if item in app_key:
                        candidate = item
                        break
            # fallback to app_key itself
            if not candidate:
                candidate = app_key
            # block
            block_app(candidate, int(SETTINGS.get("block_duration_minutes", 60)))
    except Exception:
        traceback.print_exc()

# ----------------- Notifications & goals -----------------
notified_thresholds = set()  # (date, app)
notified_goals = set()       # (date, goal_id)
last_notified_day = datetime.date.today().strftime("%Y-%m-%d")

def send_notification(title, message):
    try:
        if notification:
            notification.notify(title=title, message=message, timeout=5)
        else:
            # fallback to messagebox
            messagebox.showinfo(title, message)
    except Exception:
        try:
            messagebox.showinfo(title, message)
        except Exception:
            print(title, message)

def check_thresholds_and_goals():
    global notified_thresholds, notified_goals, last_notified_day
    today = datetime.date.today().strftime("%Y-%m-%d")
    if today != last_notified_day:
        notified_thresholds = set()
        notified_goals = set()
        last_notified_day = today

    # per-app totals today
    rows = db.query_usage_by_date(today)
    totals_by_app = {app: sec for app, sec in rows}

    # threshold notifications for unproductive
    thr_seconds = int(SETTINGS.get("usage_limit_minutes", 30)) * 60
    for app, sec in totals_by_app.items():
        key = (today, app)
        # check unproductive
        if any(k in (app or "").lower() for k in RULES["unproductive"]):
            if sec >= thr_seconds and key not in notified_thresholds:
                send_notification("Threshold crossed", f"You've spent {seconds_to_hms(sec)} on {app} today.")
                notified_thresholds.add(key)
                # also auto-block now
                maybe_auto_block(app)

    # goal checks (existing logic)
    goals = db.get_goals()
    for gid, name, target_seconds, period, active in goals:
        if not active:
            continue
        key_lower = name.lower()
        if period == 'daily':
            if 'productive' in key_lower or 'code' in key_lower or 'coding' in key_lower:
                totals = compute_productivity_for_date(today)
                current = totals['productive']
            elif 'social' in key_lower or 'unproductive' in key_lower:
                totals = compute_productivity_for_date(today)
                current = totals['unproductive']
            else:
                rows_total = db.query_usage_by_date(today)
                current = sum(sec for _, sec in rows_total)
        else:  # weekly
            end = datetime.date.today()
            start = end - datetime.timedelta(days=6)
            rows_between = db.query_usage_between(start.strftime("%Y-%m-%d"), end.strftime("%Y-%m-%d"))
            if 'productive' in key_lower or 'code' in key_lower or 'coding' in key_lower:
                current = sum(sec for app, sec in rows_between if any(k in (app or "").lower() for k in RULES["productive"]))
            elif 'social' in key_lower or 'unproductive' in key_lower:
                current = sum(sec for app, sec in rows_between if any(k in (app or "").lower() for k in RULES["unproductive"]))
            else:
                current = sum(sec for _, sec in rows_between)

        if current >= target_seconds and (today, gid) not in notified_goals:
            send_notification("Goal achieved!", f"Goal '{name}' reached: {seconds_to_hms(current)}")
            notified_goals.add((today, gid))

# ----------------- GUI App -----------------
class HabitTrackerApp:
    def __init__(self, root):
        self.root = root
        root.title("ZENTRACK — Habit Tracker (Full) with Distraction Blocker")
        root.geometry("1150x760")
        self.create_widgets()
        self._refresh_ui()
        # periodic checks every 30s
        self.root.after(30 * 1000, self._periodic_checks)

    def create_widgets(self):
        # Top row controls
        top = ttk.Frame(self.root)
        top.pack(side=tk.TOP, fill=tk.X, padx=8, pady=8)

        self.auto_var = tk.BooleanVar(value=False)
        ttk.Checkbutton(top, text="Enable Auto Tracking (best-effort)", variable=self.auto_var, command=self.toggle_auto).pack(side=tk.LEFT)

        ttk.Button(top, text="Manual Entry", command=self.manual_entry_dialog).pack(side=tk.LEFT, padx=6)
        ttk.Button(top, text="Show Today's Pie", command=self.show_today_pie).pack(side=tk.LEFT, padx=6)
        ttk.Button(top, text="Show 7-day Trend", command=lambda: self.show_trend(days=7)).pack(side=tk.LEFT, padx=6)
        ttk.Button(top, text="Show 30-day Trend", command=lambda: self.show_trend(days=30)).pack(side=tk.LEFT, padx=6)
        ttk.Button(top, text="Set Goal", command=self.set_goal_dialog).pack(side=tk.LEFT, padx=6)
        ttk.Button(top, text="Export CSV", command=self.export_csv).pack(side=tk.LEFT, padx=6)
        ttk.Button(top, text="Seed Demo Data", command=self.seed_demo_data).pack(side=tk.LEFT, padx=6)

        # Distraction blocker controls
        ttk.Separator(top, orient=tk.VERTICAL).pack(side=tk.LEFT, fill=tk.Y, padx=8)
        self.block_enable_var = tk.BooleanVar(value=True)
        ttk.Checkbutton(top, text="Enable Distraction Blocker", variable=self.block_enable_var).pack(side=tk.LEFT)
        ttk.Button(top, text="Unblock All Expired", command=self._cleanup_blocked).pack(side=tk.LEFT, padx=6)

        # Productivity score label + progress bar (right side of top)
        right_top = ttk.Frame(top)
        right_top.pack(side=tk.RIGHT)
        self.score_label = ttk.Label(right_top, text="Productivity: —", font=("TkDefaultFont", 10, "bold"))
        self.score_label.pack(anchor='e')
        self.score_bar = ttk.Progressbar(right_top, orient='horizontal', length=180, mode='determinate')
        self.score_bar.pack(anchor='e', pady=(4,0))

        # Middle area: left stats + right chart
        mid = ttk.Frame(self.root)
        mid.pack(fill=tk.BOTH, expand=True, padx=8, pady=6)

        left = ttk.Frame(mid, width=360)
        left.pack(side=tk.LEFT, fill=tk.Y, padx=(0,8))

        ttk.Label(left, text="Today's Summary", font=("TkDefaultFont", 12, "bold")).pack(pady=(0,6))
        self.today_summary = tk.Text(left, width=46, height=12, state=tk.DISABLED, wrap=tk.WORD)
        self.today_summary.pack()

        # Blocked apps list
        ttk.Label(left, text="Blocked Apps (auto/manual)", font=("TkDefaultFont", 11, "bold")).pack(pady=(8,0))
        self.blocked_list = tk.Text(left, width=46, height=6, state=tk.DISABLED)
        self.blocked_list.pack(pady=(2,6))

        # Manual block controls
        ttk.Label(left, text="Manual Block / Quick actions").pack(pady=(6,0))
        manu_frame = ttk.Frame(left)
        manu_frame.pack(fill=tk.X, pady=(2,4))
        self.manual_block_entry = ttk.Entry(manu_frame)
        self.manual_block_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=(0,6))
        ttk.Button(manu_frame, text="Block Now (1h)", command=self.manual_block_now).pack(side=tk.LEFT)

        # Goals Treeview
        ttk.Label(left, text="Goals", font=("TkDefaultFont", 11, "bold")).pack(pady=(8,0))
        self.goals_list = ttk.Treeview(left, columns=("target", "period", "progress", "active"), show="headings", height=6)
        self.goals_list.heading("target", text="Target")
        self.goals_list.heading("period", text="Period")
        self.goals_list.heading("progress", text="Progress")
        self.goals_list.heading("active", text="Active")
        self.goals_list.pack()
        self.goals_list.bind("<Double-1>", self.toggle_goal_active)

        btns_g = ttk.Frame(left)
        btns_g.pack(pady=(6,0))
        ttk.Button(btns_g, text="Remove Goal", command=self.remove_selected_goal).pack(side=tk.LEFT, padx=6)
        ttk.Button(btns_g, text="Refresh", command=self._refresh_ui).pack(side=tk.LEFT)

        # Right: chart canvas + settings
        right = ttk.Frame(mid)
        right.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        self.figure = Figure(figsize=(7,5))
        self.canvas = FigureCanvasTkAgg(self.figure, master=right)
        self.canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True)

        # Bottom: rules and settings editing
        bottom = ttk.Frame(self.root)
        bottom.pack(side=tk.BOTTOM, fill=tk.X, padx=8, pady=8)

        # Rules text
        ttk.Label(bottom, text="Productive apps (comma-separated):").pack(side=tk.LEFT)
        self.prod_entry = ttk.Entry(bottom, width=36)
        self.prod_entry.insert(0, ",".join(RULES["productive"]))
        self.prod_entry.pack(side=tk.LEFT, padx=(4,10))

        ttk.Label(bottom, text="Unproductive apps:").pack(side=tk.LEFT)
        self.unprod_entry = ttk.Entry(bottom, width=36)
        self.unprod_entry.insert(0, ",".join(RULES["unproductive"]))
        self.unprod_entry.pack(side=tk.LEFT, padx=(4,4))

        ttk.Button(bottom, text="Save Rules", command=self.save_rules).pack(side=tk.LEFT, padx=(6,0))

        # Settings button (opens settings window)
        ttk.Button(bottom, text="Settings", command=self.open_settings_window).pack(side=tk.RIGHT, padx=(6,0))

    # ---------------- UI Actions ----------------
    def toggle_auto(self):
        enabled = self.auto_var.get()
        if enabled:
            ok = tracker.start_tracking()
            if ok:
                messagebox.showinfo("Auto Tracking", "Auto-tracking started in background.")
            else:
                messagebox.showwarning("Auto Tracking", "Automatic tracking is not available on this system. Use manual entry.")
                self.auto_var.set(False)
        else:
            tracker.stop_tracking()
            messagebox.showinfo("Auto Tracking", "Auto-tracking stopped (and current session flushed).")
        self._refresh_ui()

    def manual_entry_dialog(self):
        dlg = tk.Toplevel(self.root)
        dlg.title("Manual Usage Entry")
        ttk.Label(dlg, text="App / Activity name:").pack(padx=8, pady=(8,2))
        app_e = ttk.Entry(dlg, width=40)
        app_e.pack(padx=8, pady=(0,8))
        app_e.focus()

        ttk.Label(dlg, text="Duration (minutes):").pack(padx=8, pady=(4,2))
        dur_e = ttk.Entry(dlg, width=20)
        dur_e.pack(padx=8, pady=(0,8))

        def addit():
            app = app_e.get().strip() or "manual"
            try:
                mins = float(dur_e.get().strip())
            except Exception:
                messagebox.showerror("Invalid", "Please enter numeric minutes.")
                return
            secs = int(mins * 60)
            db.add_manual_usage(app, secs)
            maybe_auto_block(app)  # check if blocking needed after adding manual
            messagebox.showinfo("Added", f"Added {seconds_to_hms(secs)} for {app}.")
            dlg.destroy()
            self._refresh_ui()

        ttk.Button(dlg, text="Add", command=addit).pack(pady=(6,12))

    def show_today_pie(self):
        today = datetime.date.today().strftime("%Y-%m-%d")
        rows = db.query_usage_by_date(today)
        labels = []
        values = []
        for app, sec in rows:
            labels.append(app or "unknown")
            values.append(sec)
        if not values:
            messagebox.showinfo("No data", "No usage data for today.")
            return
        self.figure.clf()
        ax = self.figure.add_subplot(111)
        # autopct prints total minutes
        ax.pie(values, labels=labels, autopct=lambda pct: f"{int(round(pct*sum(values)/100.0))//60}m", startangle=90)
        ax.set_title(f"Time distribution — {today}")
        self.canvas.draw()

    def show_trend(self, days=7):
        data = db.query_daily_totals(days=days)
        dates = [d[0] for d in data]
        prod = [d[1]/3600.0 for d in data]
        unprod = [d[2]/3600.0 for d in data]
        neutral = [d[3]/3600.0 for d in data]

        self.figure.clf()
        ax = self.figure.add_subplot(111)
        ax.plot(dates, prod, marker='o', label="Productive (hrs)")
        ax.plot(dates, unprod, marker='o', label="Unproductive (hrs)")
        ax.plot(dates, neutral, marker='o', label="Neutral (hrs)")
        ax.set_xticks(dates[::max(1, len(dates)//10)])
        ax.set_xticklabels(dates[::max(1, len(dates)//10)], rotation=45, ha="right")
        ax.set_title(f"{days}-day usage trends")
        ax.legend()
        self.canvas.draw()

    def set_goal_dialog(self):
        dlg = tk.Toplevel(self.root)
        dlg.title("Set Goal")
        ttk.Label(dlg, text="Goal name (e.g. 'Coding' or 'Social <2h')").pack(padx=8, pady=(8,2))
        name_e = ttk.Entry(dlg, width=40)
        name_e.pack(padx=8, pady=(0,8))
        ttk.Label(dlg, text="Target duration (minutes):").pack(padx=8, pady=(4,2))
        dur_e = ttk.Entry(dlg, width=20)
        dur_e.pack(padx=8, pady=(0,8))
        ttk.Label(dlg, text="Period:").pack(padx=8, pady=(4,2))
        period_var = tk.StringVar(value="daily")
        ttk.Radiobutton(dlg, text="Daily", variable=period_var, value="daily").pack(anchor='w', padx=8)
        ttk.Radiobutton(dlg, text="Weekly", variable=period_var, value="weekly").pack(anchor='w', padx=8)

        def add_goal():
            name = name_e.get().strip() or "Unnamed"
            try:
                mins = float(dur_e.get().strip())
            except Exception:
                messagebox.showerror("Invalid", "Enter numeric minutes.")
                return
            secs = int(mins * 60)
            db.add_goal(name, secs, period_var.get())
            messagebox.showinfo("Added", f"Goal '{name}' added.")
            dlg.destroy()
            self._refresh_ui()

        ttk.Button(dlg, text="Add Goal", command=add_goal).pack(pady=(8,12))

    def export_csv(self):
        path = filedialog.asksaveasfilename(
            defaultextension=".csv",
            filetypes=[("CSV files", "*.csv")]
        )
        if not path:
            return
        c = db.conn.cursor()
        c.execute("SELECT app_name, start_ts, end_ts, duration_seconds, date FROM usage ORDER BY date, start_ts")
        rows = c.fetchall()

        import csv
        try:
            with open(path, "w", newline="", encoding="utf-8") as f:
                writer = csv.writer(f)
                writer.writerow(["App Name", "Start Time", "End Time", "Duration (seconds)", "Date"])
                for app, start_ts, end_ts, duration, date in rows:
                    start_str = datetime.datetime.fromtimestamp(start_ts).strftime("%Y-%m-%d %H:%M:%S")
                    end_str = datetime.datetime.fromtimestamp(end_ts).strftime("%Y-%m-%d %H:%M:%S")
                    writer.writerow([app, start_str, end_str, duration, date])
            messagebox.showinfo("Export Complete", f"Data exported to {path}")
        except Exception as e:
            messagebox.showerror("Export Failed", f"Could not export CSV: {e}")

    def seed_demo_data(self):
        """Insert demo data for last 14 days (useful to test charts)."""
        apps = ["Code - VSCode", "YouTube", "Word", "Instagram", "Terminal", "Spotify", "Zoom", "Facebook", "Chrome"]
        now = datetime.datetime.now()
        for days_ago in range(0, 14):
            date = (now - datetime.timedelta(days=days_ago)).date()
            # create 3-6 random sessions that day
            for _ in range(random.randint(3,6)):
                app = random.choice(apps)
                # pick a start hour and duration (minutes)
                start_hour = random.randint(8, 22)
                start_minute = random.randint(0, 59)
                dur_min = random.choice([5, 10, 15, 20, 30, 45, 60, 90])
                start_dt = datetime.datetime.combine(date, datetime.time(start_hour, start_minute))
                start_ts = int(start_dt.timestamp())
                end_ts = start_ts + dur_min * 60
                db.add_usage(app, start_ts, end_ts)
        messagebox.showinfo("Demo Data", "Seeded 14 days of demo usage. Refreshing UI.")
        self._refresh_ui()

    def save_rules(self):
        prod_text = self.prod_entry.get().strip()
        unprod_text = self.unprod_entry.get().strip()
        prod_list = [s.strip().lower() for s in prod_text.split(",") if s.strip()]
        unprod_list = [s.strip().lower() for s in unprod_text.split(",") if s.strip()]
        RULES["productive"] = prod_list
        RULES["unproductive"] = unprod_list
        save_rules_to_file(prod_list, unprod_list)
        messagebox.showinfo("Saved", "Rules saved.")
        self._refresh_ui()

    def _refresh_ui(self):
        # update today's summary and productivity score and goals
        today = datetime.date.today().strftime("%Y-%m-%d")
        rows = db.query_usage_by_date(today)
        totals = {"productive": 0, "unproductive": 0, "neutral": 0}
        summary_lines = []
        total_tracked = 0
        for app, sec in rows:
            cls = classify_app(app)
            totals[cls] += sec
            total_tracked += sec
            summary_lines.append(f"{app or 'unknown'} — {seconds_to_hms(sec)} ({cls})")
        # update summary text
        self.today_summary.config(state=tk.NORMAL)
        self.today_summary.delete("1.0", tk.END)
        if summary_lines:
            self.today_summary.insert(tk.END, "\n".join(summary_lines))
        else:
            self.today_summary.insert(tk.END, "No usage logged for today yet.")
        self.today_summary.config(state=tk.DISABLED)

        # productivity score (productive / total tracked * 100)
        if total_tracked > 0:
            pct = int((totals["productive"] / total_tracked) * 100)
        else:
            pct = 0
        self.score_bar['value'] = pct
        self.score_label.config(text=f"Productivity: {pct}%  (prod {seconds_to_hms(totals['productive'])} / total {seconds_to_hms(total_tracked)})")

        # update goals table
        for row in self.goals_list.get_children():
            self.goals_list.delete(row)
        goals = db.get_goals()
        for gid, name, target_seconds, period, active in goals:
            # compute current
            if period == 'daily':
                if 'productive' in name.lower() or 'code' in name.lower() or 'coding' in name.lower():
                    cur = compute_productivity_for_date(today)['productive']
                elif 'social' in name.lower() or 'unproductive' in name.lower():
                    cur = compute_productivity_for_date(today)['unproductive']
                else:
                    rows_total = db.query_usage_by_date(today)
                    cur = sum(sec for _, sec in rows_total)
            else:  # weekly
                end = datetime.date.today()
                start = end - datetime.timedelta(days=6)
                rows_between = db.query_usage_between(start.strftime("%Y-%m-%d"), end.strftime("%Y-%m-%d"))
                if 'productive' in name.lower() or 'code' in name.lower() or 'coding' in name.lower():
                    cur = sum(sec for app, sec in rows_between if any(k in (app or "").lower() for k in RULES["productive"]))
                elif 'social' in name.lower() or 'unproductive' in name.lower():
                    cur = sum(sec for app, sec in rows_between if any(k in (app or "").lower() for k in RULES["unproductive"]))
                else:
                    cur = sum(sec for _, sec in rows_between)
            pct_goal = int(min(100, (cur / target_seconds) * 100)) if target_seconds > 0 else 0
            prog_text = f"{pct_goal}% ({seconds_to_hms(cur)}/{seconds_to_hms(target_seconds)})"
            self.goals_list.insert("", tk.END, iid=str(gid), values=(seconds_to_hms(target_seconds), period, prog_text, "Yes" if active else "No"))

        # update blocked list UI
        self.update_blocked_list()

        # refresh chart with 7-day by default (non-blocking)
        try:
            self.show_trend(days=7)
        except Exception:
            pass

    def toggle_goal_active(self, event):
        sel = self.goals_list.selection()
        if not sel:
            return
        gid = int(sel[0])
        # get current active
        cur = db.conn.cursor()
        cur.execute("SELECT active FROM goals WHERE id = ?", (gid,))
        val = cur.fetchone()
        if not val:
            return
        new_active = 0 if val[0] else 1
        db.toggle_goal(gid, new_active)
        self._refresh_ui()

    def remove_selected_goal(self):
        sel = self.goals_list.selection()
        if not sel:
            messagebox.showinfo("Remove Goal", "Select a goal first.")
            return
        gid = int(sel[0])
        if messagebox.askyesno("Remove Goal", "Are you sure you want to remove the selected goal?"):
            db.remove_goal(gid)
            self._refresh_ui()

    def _periodic_checks(self):
        # refresh UI and run threshold/goal checks
        self._refresh_ui()
        check_thresholds_and_goals()
        # schedule again
        self.root.after(30 * 1000, self._periodic_checks)

    # ---------------- Block UI actions ----------------
    def update_blocked_list(self):
        self.blocked_list.config(state=tk.NORMAL)
        self.blocked_list.delete("1.0", tk.END)
        now = time.time()
        with blocked_apps_lock:
            for app, unblock_time in list(blocked_apps.items()):
                mins_left = int((unblock_time - now) / 60)
                if mins_left > 0:
                    self.blocked_list.insert(tk.END, f"{app} → {mins_left} min(s) left\n")
        self.blocked_list.config(state=tk.DISABLED)

    def manual_block_now(self):
        app_text = self.manual_block_entry.get().strip()
        if not app_text:
            messagebox.showerror("Error", "Enter an app process name (e.g. chrome.exe) or partial name.")
            return
        # normalize to lowercase
        key = app_text.lower()
        block_app(key, int(SETTINGS.get("block_duration_minutes", 60)))
        messagebox.showinfo("Blocked", f"{key} blocked for {SETTINGS.get('block_duration_minutes', 60)} minutes.")
        self._refresh_ui()

    def _cleanup_blocked(self):
        # remove expired entries (functionally done by is_blocked but clean UI)
        with blocked_apps_lock:
            for k in list(blocked_apps.keys()):
                if time.time() >= blocked_apps[k]:
                    del blocked_apps[k]
        self._refresh_ui()

    # ---------------- Settings Window ----------------
    def open_settings_window(self):
        sdlg = tk.Toplevel(self.root)
        sdlg.title("Settings")
        sdlg.geometry("560x360")

        # Unproductive apps list
        ttk.Label(sdlg, text="Unproductive app process names (one per line, e.g. chrome.exe)").pack(anchor='w', padx=8, pady=(8,2))
        apps_text = tk.Text(sdlg, height=8)
        apps_text.pack(fill=tk.BOTH, padx=8)
        apps_text.delete("1.0", tk.END)
        for a in SETTINGS.get("unproductive_apps", []):
            apps_text.insert(tk.END, a + "\n")

        # Threshold
        frm = ttk.Frame(sdlg)
        frm.pack(fill=tk.X, padx=8, pady=(8,4))
        ttk.Label(frm, text="Usage limit (minutes) before automatic block:").pack(side=tk.LEFT)
        limit_entry = ttk.Entry(frm, width=8)
        limit_entry.pack(side=tk.LEFT, padx=(6,0))
        limit_entry.insert(0, str(SETTINGS.get("usage_limit_minutes", 30)))

        ttk.Label(frm, text="Block duration (minutes):").pack(side=tk.LEFT, padx=(12,0))
        blockdur_entry = ttk.Entry(frm, width=8)
        blockdur_entry.pack(side=tk.LEFT, padx=(6,0))
        blockdur_entry.insert(0, str(SETTINGS.get("block_duration_minutes", 60)))

        def save_and_close():
            text_val = apps_text.get("1.0", tk.END).strip().splitlines()
            apps_list = [line.strip() for line in text_val if line.strip()]
            try:
                SETTINGS["usage_limit_minutes"] = int(limit_entry.get().strip())
                SETTINGS["block_duration_minutes"] = int(blockdur_entry.get().strip())
            except Exception:
                messagebox.showerror("Error", "Please enter valid numbers for minutes fields.")
                return
            SETTINGS["unproductive_apps"] = apps_list
            # Update RULES["unproductive"] for classification too
            RULES["unproductive"] = [s.lower() for s in apps_list]
            save_settings(SETTINGS)
            save_rules_to_file(RULES["productive"], RULES["unproductive"])
            messagebox.showinfo("Saved", "Settings saved.")
            sdlg.destroy()
            self._refresh_ui()

        ttk.Button(sdlg, text="Save Settings", command=save_and_close).pack(pady=(6,10))

# ----------------- Main Execution -----------------
def ensure_db_created():
    # db is already initialized at import time
    pass

if __name__ == "__main__":
    ensure_db_created()

    # Start blocked-app monitor thread
    t_block = threading.Thread(target=monitor_blocked_apps_thread, daemon=True)
    t_block.start()

    # Start tkinter app
    root = tk.Tk()
    app = HabitTrackerApp(root)
    root.mainloop()

