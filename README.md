from pathlib import Path
import json, shutil, zipfile, textwrap, os

base = Path("/mnt/data/ServerHealthMonitor2")
templates = base / "templates"
static = base / "static"
templates.mkdir(parents=True, exist_ok=True)
static.mkdir(parents=True, exist_ok=True)

app_py = '''from flask import Flask, render_template, jsonify, request
import psutil
import platform
import socket
import os
import json
from datetime import datetime

app = Flask(__name__)

SETTINGS_FILE = "settings.json"

DEFAULT_SETTINGS = {
    "website_name": "ServerHealthMonitor2",
    "theme": "dark",
    "main_color": "blue",
    "refresh_time": 5,
    "cpu_limit": 70,
    "ram_limit": 70,
    "disk_limit": 80,
    "show_dashboard": True,
    "show_system": True,
    "show_cpu": True,
    "show_ram": True,
    "show_disk": True,
    "show_network": True,
    "show_processes": True,
    "show_os": True,
    "show_alerts": True,
    "show_reports": True
}

def save_settings(settings):
    with open(SETTINGS_FILE, "w", encoding="utf-8") as file:
        json.dump(settings, file, indent=4)

def load_settings():
    if not os.path.exists(SETTINGS_FILE):
        save_settings(DEFAULT_SETTINGS)
        return DEFAULT_SETTINGS.copy()

    try:
        with open(SETTINGS_FILE, "r", encoding="utf-8") as file:
            saved = json.load(file)
        settings = DEFAULT_SETTINGS.copy()
        settings.update(saved)
        return settings
    except (OSError, json.JSONDecodeError):
        return DEFAULT_SETTINGS.copy()

def get_ip():
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        sock.connect(("8.8.8.8", 80))
        ip = sock.getsockname()[0]
        sock.close()
        return ip
    except OSError:
        try:
            return socket.gethostbyname(socket.gethostname())
        except OSError:
            return "Unavailable"

def get_status(value, warning, critical):
    if value >= critical:
        return "CRITICAL"
    if value >= warning:
        return "WARNING"
    return "HEALTHY"

@app.route("/")
def home():
    return render_template("index.html")

@app.route("/admin")
def admin():
    return render_template("admin.html")

@app.route("/api/system")
def system_data():
    settings = load_settings()
    cpu = psutil.cpu_percent(interval=0.3)
    memory = psutil.virtual_memory()
    disk = psutil.disk_usage(os.path.abspath(os.sep))

    return jsonify({
        "computer": socket.gethostname(),
        "os": platform.system(),
        "os_release": platform.release(),
        "processor": platform.processor() or "Unavailable",
        "cpu": cpu,
        "cpu_cores": psutil.cpu_count(logical=True),
        "ram": memory.percent,
        "ram_total": round(memory.total / (1024 ** 3), 2),
        "disk": disk.percent,
        "disk_total": round(disk.total / (1024 ** 3), 2),
        "disk_free": round(disk.free / (1024 ** 3), 2),
        "ip": get_ip(),
        "cpu_status": get_status(cpu, int(settings["cpu_limit"]), 90),
        "ram_status": get_status(memory.percent, int(settings["ram_limit"]), 90),
        "disk_status": get_status(disk.percent, int(settings["disk_limit"]), 95),
        "date_time": datetime.now().strftime("%d/%m/%Y %H:%M:%S")
    })

@app.route("/api/processes")
def processes():
    items = []
    for process in psutil.process_iter(["pid", "name", "memory_percent"]):
        try:
            items.append({
                "pid": process.info["pid"],
                "name": process.info["name"],
                "memory": round(process.info["memory_percent"], 2)
            })
            if len(items) >= 50:
                break
        except (psutil.NoSuchProcess, psutil.AccessDenied):
            pass
    return jsonify(items)

@app.route("/api/settings", methods=["GET", "POST"])
def settings_api():
    if request.method == "GET":
        return jsonify(load_settings())

    incoming = request.get_json(silent=True) or {}
    settings = load_settings()

    for key in DEFAULT_SETTINGS:
        if key in incoming:
            settings[key] = incoming[key]

    settings["refresh_time"] = max(2, min(60, int(settings["refresh_time"])))
    settings["cpu_limit"] = max(1, min(100, int(settings["cpu_limit"])))
    settings["ram_limit"] = max(1, min(100, int(settings["ram_limit"])))
    settings["disk_limit"] = max(1, min(100, int(settings["disk_limit"])))

    save_settings(settings)

    return jsonify({
        "success": True,
        "message": "Settings saved successfully",
        "settings": settings
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
'''

index_html = '''<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ServerHealthMonitor2</title>
  <link rel="stylesheet" href="/static/style.css">
</head>
<body class="theme-dark color-blue">
<div class="layout">

  <aside class="sidebar">
    <div class="logo-area">
      <img src="/static/logo.png" alt="SERVERHub logo">
    </div>
    <div class="sidebar-title">SERVER MONITOR</div>

    <button class="nav-btn" data-setting="show_dashboard" onclick="showPage('dashboard')">Dashboard</button>
    <button class="nav-btn" data-setting="show_system" onclick="showPage('system')">System Info</button>
    <button class="nav-btn" data-setting="show_cpu" onclick="showPage('cpu')">CPU Monitor</button>
    <button class="nav-btn" data-setting="show_ram" onclick="showPage('ram')">RAM Monitor</button>
    <button class="nav-btn" data-setting="show_disk" onclick="showPage('disk')">Disk Monitor</button>
    <button class="nav-btn" data-setting="show_network" onclick="showPage('network')">Network Monitor</button>
    <button class="nav-btn" data-setting="show_processes" onclick="showProcesses()">Processes</button>
    <button class="nav-btn" data-setting="show_os" onclick="showPage('os')">Operating Systems</button>
    <button class="nav-btn" data-setting="show_alerts" onclick="showPage('alerts')">Alerts</button>
    <button class="nav-btn" data-setting="show_reports" onclick="showPage('reports')">Reports</button>

    <a class="admin-link" href="/admin">Admin Control</a>
  </aside>

  <main class="main">
    <header class="topbar">
      <div>
        <h1 id="website-title">ServerHealthMonitor2</h1>
        <p>Smart Server Health & Performance Monitoring</p>
      </div>
      <button class="primary-btn" onclick="loadSystemData()">Refresh</button>
    </header>

    <section id="dashboard" class="page active">
      <div class="page-heading">
        <h2>System Dashboard</h2>
        <p>Live system health overview</p>
      </div>

      <div class="cards">
        <div class="card">
          <span>CPU Usage</span>
          <h2 id="cpu-value">0%</h2>
          <div class="progress"><div id="cpu-bar"></div></div>
          <p id="cpu-status">Loading...</p>
        </div>

        <div class="card">
          <span>RAM Usage</span>
          <h2 id="ram-value">0%</h2>
          <div class="progress"><div id="ram-bar"></div></div>
          <p id="ram-status">Loading...</p>
        </div>

        <div class="card">
          <span>Disk Usage</span>
          <h2 id="disk-value">0%</h2>
          <div class="progress"><div id="disk-bar"></div></div>
          <p id="disk-status">Loading...</p>
        </div>
      </div>

      <div class="details">
        <div class="detail-card"><h3>Operating System</h3><p id="os-value">Loading...</p></div>
        <div class="detail-card"><h3>Computer Name</h3><p id="computer-value">Loading...</p></div>
        <div class="detail-card"><h3>IP Address</h3><p id="ip-value">Loading...</p></div>
      </div>
    </section>

    <section id="system" class="page">
      <h2>System Information</h2>
      <div class="info-panel">
        <p><strong>Computer:</strong> <span id="system-computer"></span></p>
        <p><strong>Operating System:</strong> <span id="system-os"></span></p>
        <p><strong>OS Version:</strong> <span id="system-version"></span></p>
        <p><strong>Processor:</strong> <span id="system-processor"></span></p>
        <p><strong>CPU Cores:</strong> <span id="system-cores"></span></p>
        <p><strong>Total RAM:</strong> <span id="system-ram-total"></span></p>
        <p><strong>IP Address:</strong> <span id="system-ip"></span></p>
      </div>
    </section>

    <section id="cpu" class="page">
      <h2>CPU Monitor</h2>
      <div class="big-monitor"><h1 id="cpu-large">0%</h1><p>Current CPU Usage</p></div>
    </section>

    <section id="ram" class="page">
      <h2>RAM Monitor</h2>
      <div class="big-monitor"><h1 id="ram-large">0%</h1><p>Current RAM Usage</p></div>
    </section>

    <section id="disk" class="page">
      <h2>Disk Monitor</h2>
      <div class="big-monitor">
        <h1 id="disk-large">0%</h1>
        <p>Current Disk Usage</p>
        <p>Total: <span id="disk-total"></span></p>
        <p>Free: <span id="disk-free"></span></p>
      </div>
    </section>

    <section id="network" class="page">
      <h2>Network Monitor</h2>
      <div class="info-panel">
        <p><strong>Computer:</strong> <span id="network-computer"></span></p>
        <p><strong>IP Address:</strong> <span id="network-ip"></span></p>
        <p><strong>Status:</strong> Connected</p>
      </div>
    </section>

    <section id="processes" class="page">
      <div class="section-row">
        <h2>Running Processes</h2>
        <button class="primary-btn" onclick="loadProcesses()">Refresh Processes</button>
      </div>
      <div id="process-list" class="process-list"></div>
    </section>

    <section id="os" class="page">
      <h2>Operating Systems</h2>
      <div class="os-grid">
        <div>Windows</div>
        <div>macOS</div>
        <div>Linux</div>
        <div>Android</div>
        <div>iOS</div>
        <div>iPadOS</div>
        <div>watchOS</div>
        <div>tvOS</div>
      </div>
    </section>

    <section id="alerts" class="page">
      <h2>Alerts</h2>
      <div class="info-panel"><p id="alert-text">No active alerts.</p></div>
    </section>

    <section id="reports" class="page">
      <h2>System Report</h2>
      <div class="info-panel">
        <p><strong>Date:</strong> <span id="report-date"></span></p>
        <p><strong>CPU:</strong> <span id="report-cpu"></span></p>
        <p><strong>RAM:</strong> <span id="report-ram"></span></p>
        <p><strong>Disk:</strong> <span id="report-disk"></span></p>
        <p><strong>Operating System:</strong> <span id="report-os"></span></p>
      </div>
    </section>
  </main>
</div>

<script>
let refreshTimer = null;
let currentSettings = null;

function showPage(id) {
  document.querySelectorAll(".page").forEach(page => page.classList.remove("active"));
  document.getElementById(id).classList.add("active");
}

function statusClass(status) {
  return status === "CRITICAL" ? "critical" : status === "WARNING" ? "warning" : "healthy";
}

async function loadSettings() {
  const response = await fetch("/api/settings");
  const settings = await response.json();
  currentSettings = settings;

  document.title = settings.website_name;
  document.getElementById("website-title").innerText = settings.website_name;

  document.body.className = `theme-${settings.theme} color-${settings.main_color}`;

  document.querySelectorAll("[data-setting]").forEach(button => {
    const key = button.dataset.setting;
    button.style.display = settings[key] ? "block" : "none";
  });

  if (refreshTimer) clearInterval(refreshTimer);
  refreshTimer = setInterval(loadSystemData, Number(settings.refresh_time) * 1000);
}

async function loadSystemData() {
  const response = await fetch("/api/system");
  const data = await response.json();

  document.getElementById("cpu-value").innerText = data.cpu + "%";
  document.getElementById("ram-value").innerText = data.ram + "%";
  document.getElementById("disk-value").innerText = data.disk + "%";

  document.getElementById("cpu-bar").style.width = data.cpu + "%";
  document.getElementById("ram-bar").style.width = data.ram + "%";
  document.getElementById("disk-bar").style.width = data.disk + "%";

  const cpuStatus = document.getElementById("cpu-status");
  cpuStatus.innerText = data.cpu_status;
  cpuStatus.className = statusClass(data.cpu_status);

  const ramStatus = document.getElementById("ram-status");
  ramStatus.innerText = data.ram_status;
  ramStatus.className = statusClass(data.ram_status);

  const diskStatus = document.getElementById("disk-status");
  diskStatus.innerText = data.disk_status;
  diskStatus.className = statusClass(data.disk_status);

  document.getElementById("os-value").innerText = data.os + " " + data.os_release;
  document.getElementById("computer-value").innerText = data.computer;
  document.getElementById("ip-value").innerText = data.ip;

  document.getElementById("system-computer").innerText = data.computer;
  document.getElementById("system-os").innerText = data.os;
  document.getElementById("system-version").innerText = data.os_release;
  document.getElementById("system-processor").innerText = data.processor;
  document.getElementById("system-cores").innerText = data.cpu_cores;
  document.getElementById("system-ram-total").innerText = data.ram_total + " GB";
  document.getElementById("system-ip").innerText = data.ip;

  document.getElementById("cpu-large").innerText = data.cpu + "%";
  document.getElementById("ram-large").innerText = data.ram + "%";
  document.getElementById("disk-large").innerText = data.disk + "%";
  document.getElementById("disk-total").innerText = data.disk_total + " GB";
  document.getElementById("disk-free").innerText = data.disk_free + " GB";

  document.getElementById("network-computer").innerText = data.computer;
  document.getElementById("network-ip").innerText = data.ip;

  const alerts = [];
  if (data.cpu_status !== "HEALTHY") alerts.push("CPU: " + data.cpu_status);
  if (data.ram_status !== "HEALTHY") alerts.push("RAM: " + data.ram_status);
  if (data.disk_status !== "HEALTHY") alerts.push("Disk: " + data.disk_status);

  document.getElementById("alert-text").innerText =
    alerts.length ? alerts.join(" | ") : "No active alerts. System is healthy.";

  document.getElementById("report-date").innerText = data.date_time;
  document.getElementById("report-cpu").innerText = data.cpu + "%";
  document.getElementById("report-ram").innerText = data.ram + "%";
  document.getElementById("report-disk").innerText = data.disk + "%";
  document.getElementById("report-os").innerText = data.os + " " + data.os_release;
}

async function showProcesses() {
  showPage("processes");
  await loadProcesses();
}

async function loadProcesses() {
  const response = await fetch("/api/processes");
  const processes = await response.json();
  const list = document.getElementById("process-list");
  list.innerHTML = "";

  processes.forEach(process => {
    const item = document.createElement("div");
    item.className = "process-item";
    item.innerHTML = `<strong>${process.name}</strong><span>PID: ${process.pid} | Memory: ${process.memory}%</span>`;
    list.appendChild(item);
  });
}

async function startApp() {
  await loadSettings();
  await loadSystemData();
}

startApp();
</script>
</body>
</html>
'''

admin_html = '''<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Admin Control</title>
  <link rel="stylesheet" href="/static/style.css">
</head>
<body class="theme-dark color-blue">

<div class="admin-shell">
  <div class="admin-header">
    <div>
      <h1>SERVERHub Admin Control</h1>
      <p>Control the website settings from here.</p>
    </div>
    <a class="back-link" href="/">Back to Website</a>
  </div>

  <div class="admin-grid">
    <div class="admin-card">
      <h2>Website</h2>

      <label>Website Name</label>
      <input id="website_name" type="text">

      <label>Theme</label>
      <select id="theme">
        <option value="dark">Dark</option>
        <option value="light">Light</option>
      </select>

      <label>Main Color</label>
      <select id="main_color">
        <option value="blue">Blue</option>
        <option value="green">Green</option>
        <option value="purple">Purple</option>
      </select>

      <label>Refresh Time (seconds)</label>
      <input id="refresh_time" type="number" min="2" max="60">
    </div>

    <div class="admin-card">
      <h2>Alert Limits</h2>

      <label>CPU Warning %</label>
      <input id="cpu_limit" type="number" min="1" max="100">

      <label>RAM Warning %</label>
      <input id="ram_limit" type="number" min="1" max="100">

      <label>Disk Warning %</label>
      <input id="disk_limit" type="number" min="1" max="100">
    </div>

    <div class="admin-card feature-card">
      <h2>Show / Hide Features</h2>

      <label><input id="show_dashboard" type="checkbox"> Dashboard</label>
      <label><input id="show_system" type="checkbox"> System Info</label>
      <label><input id="show_cpu" type="checkbox"> CPU Monitor</label>
      <label><input id="show_ram" type="checkbox"> RAM Monitor</label>
      <label><input id="show_disk" type="checkbox"> Disk Monitor</label>
      <label><input id="show_network" type="checkbox"> Network Monitor</label>
      <label><input id="show_processes" type="checkbox"> Processes</label>
      <label><input id="show_os" type="checkbox"> Operating Systems</label>
      <label><input id="show_alerts" type="checkbox"> Alerts</label>
      <label><input id="show_reports" type="checkbox"> Reports</label>
    </div>
  </div>

  <button class="save-btn" onclick="saveSettings()">Save Changes</button>
  <p id="save-message"></p>
</div>

<script>
const fields = [
  "website_name", "theme", "main_color", "refresh_time",
  "cpu_limit", "ram_limit", "disk_limit",
  "show_dashboard", "show_system", "show_cpu", "show_ram",
  "show_disk", "show_network", "show_processes", "show_os",
  "show_alerts", "show_reports"
];

async function loadSettings() {
  const response = await fetch("/api/settings");
  const settings = await response.json();

  fields.forEach(id => {
    const element = document.getElementById(id);
    if (element.type === "checkbox") {
      element.checked = Boolean(settings[id]);
    } else {
      element.value = settings[id];
    }
  });

  document.body.className = `theme-${settings.theme} color-${settings.main_color}`;
}

async function saveSettings() {
  const settings = {};

  fields.forEach(id => {
    const element = document.getElementById(id);
    settings[id] = element.type === "checkbox" ? element.checked : element.value;
  });

  const response = await fetch("/api/settings", {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify(settings)
  });

  const result = await response.json();
  document.getElementById("save-message").innerText = result.message || "Saved";
  await loadSettings();
}

loadSettings();
</script>
</body>
</html>
'''

style_css = '''* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

:root {
  --bg: #0f172a;
  --panel: #1e293b;
  --panel-2: #111827;
  --text: #f8fafc;
  --muted: #94a3b8;
  --accent: #2563eb;
  --success: #22c55e;
  --warning: #f59e0b;
  --danger: #ef4444;
  --border: #334155;
}

body {
  font-family: Arial, sans-serif;
  background: var(--bg);
  color: var(--text);
  min-height: 100vh;
}

body.color-green {
  --accent: #16a34a;
}

body.color-purple {
  --accent: #7c3aed;
}

body.theme-light {
  --bg: #f1f5f9;
  --panel: #ffffff;
  --panel-2: #e2e8f0;
  --text: #0f172a;
  --muted: #64748b;
  --border: #cbd5e1;
}

.layout {
  display: flex;
  min-height: 100vh;
}

.sidebar {
  width: 240px;
  background: var(--panel-2);
  padding: 22px 15px;
  position: fixed;
  left: 0;
  top: 0;
  bottom: 0;
  overflow-y: auto;
}

.logo-area {
  text-align: center;
  margin-bottom: 20px;
}

.logo-area img {
  width: 165px;
  max-height: 72px;
  object-fit: contain;
}

.sidebar-title {
  color: var(--muted);
  font-size: 12px;
  font-weight: bold;
  margin: 10px 8px 14px;
}

.nav-btn,
.admin-link {
  display: block;
  width: 100%;
  border: 0;
  background: transparent;
  color: var(--text);
  text-align: left;
  padding: 12px 14px;
  margin-bottom: 7px;
  border-radius: 10px;
  text-decoration: none;
  cursor: pointer;
  font-size: 14px;
}

.nav-btn:hover,
.admin-link:hover {
  background: var(--accent);
  color: white;
}

.admin-link {
  margin-top: 18px;
  border: 1px solid var(--border);
}

.main {
  flex: 1;
  margin-left: 240px;
}

.topbar {
  background: var(--panel);
  padding: 20px 30px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid var(--border);
}

.topbar h1 {
  font-size: 24px;
}

.topbar p,
.page-heading p {
  color: var(--muted);
  margin-top: 5px;
}

.primary-btn,
.save-btn,
.back-link {
  background: var(--accent);
  color: white;
  border: 0;
  padding: 11px 18px;
  border-radius: 10px;
  cursor: pointer;
  text-decoration: none;
  display: inline-block;
}

.page {
  display: none;
  padding: 30px;
}

.page.active {
  display: block;
}

.page-heading {
  margin-bottom: 22px;
}

.cards,
.details {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 18px;
}

.details {
  margin-top: 18px;
}

.card,
.detail-card,
.info-panel,
.big-monitor,
.admin-card {
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 16px;
  padding: 22px;
}

.card span {
  color: var(--muted);
}

.card h2 {
  font-size: 34px;
  margin: 10px 0;
}

.progress {
  height: 10px;
  background: var(--border);
  border-radius: 999px;
  overflow: hidden;
  margin-bottom: 10px;
}

.progress div {
  height: 100%;
  width: 0%;
  background: var(--accent);
  transition: width .4s ease;
}

.healthy {
  color: var(--success);
  font-weight: bold;
}

.warning {
  color: var(--warning);
  font-weight: bold;
}

.critical {
  color: var(--danger);
  font-weight: bold;
}

.info-panel {
  margin-top: 20px;
}

.info-panel p {
  margin-bottom: 13px;
  line-height: 1.5;
}

.big-monitor {
  max-width: 560px;
  margin-top: 20px;
  text-align: center;
}

.big-monitor h1 {
  font-size: 70px;
  color: var(--accent);
  margin-bottom: 10px;
}

.os-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 15px;
  margin-top: 20px;
}

.os-grid div {
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 25px;
  text-align: center;
  font-weight: bold;
}

.section-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
}

.process-list {
  margin-top: 20px;
}

.process-item {
  display: flex;
  justify-content: space-between;
  gap: 15px;
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 12px 15px;
  margin-bottom: 8px;
}

.process-item span {
  color: var(--muted);
}

.admin-shell {
  max-width: 1100px;
  margin: auto;
  padding: 30px 20px 50px;
}

.admin-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 15px;
  margin-bottom: 25px;
}

.admin-header p {
  color: var(--muted);
  margin-top: 6px;
}

.admin-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 18px;
}

.admin-card h2 {
  margin-bottom: 18px;
}

.admin-card label {
  display: block;
  margin: 12px 0 7px;
}

.admin-card input[type="text"],
.admin-card input[type="number"],
.admin-card select {
  width: 100%;
  padding: 11px;
  border-radius: 9px;
  border: 1px solid var(--border);
  background: var(--bg);
  color: var(--text);
}

.feature-card {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: repeat(2, 1fr);
}

.feature-card h2 {
  grid-column: 1 / -1;
}

.feature-card label {
  background: var(--bg);
  border: 1px solid var(--border);
  padding: 12px;
  border-radius: 9px;
  margin: 5px;
}

.save-btn {
  margin-top: 20px;
  font-size: 16px;
}

#save-message {
  margin-top: 12px;
  color: var(--success);
}

@media (max-width: 800px) {
  .sidebar {
    position: static;
    width: 100%;
  }

  .layout {
    display: block;
  }

  .main {
    margin-left: 0;
  }

  .cards,
  .details {
    grid-template-columns: 1fr;
  }

  .os-grid {
    grid-template-columns: repeat(2, 1fr);
  }

  .nav-btn,
  .admin-link {
    text-align: center;
  }

  .topbar,
  .admin-header,
  .section-row {
    flex-direction: column;
    align-items: stretch;
  }

  .admin-grid {
    grid-template-columns: 1fr;
  }

  .feature-card {
    grid-column: auto;
    grid-template-columns: 1fr;
  }

  .feature-card h2 {
    grid-column: auto;
  }

  .page {
    padding: 18px;
  }
}
'''

requirements_txt = '''Flask==3.1.2
psutil==7.0.0
'''

settings_json = json.dumps({
    "website_name": "ServerHealthMonitor2",
    "theme": "dark",
    "main_color": "blue",
    "refresh_time": 5,
    "cpu_limit": 70,
    "ram_limit": 70,
    "disk_limit": 80,
    "show_dashboard": True,
    "show_system": True,
    "show_cpu": True,
    "show_ram": True,
    "show_disk": True,
    "show_network": True,
    "show_processes": True,
    "show_os": True,
    "show_alerts": True,
    "show_reports": True
}, indent=4)

readme = '''# ServerHealthMonitor2

A Flask-based server health monitoring dashboard with an Admin Control panel.

## Features

- Dashboard
- System information
- CPU monitor
- RAM monitor
- Disk monitor
- Network information
- Running processes
- Operating systems page
- Alerts
- Reports
- Admin control panel
- Responsive design for Windows, Android, and iOS browsers

## Run locally

```bash
pip install -r requirements.txt
python app.py
