# ...existing code...
import json
import threading
import time
from pathlib import Path
import tkinter as tk
from tkinter import ttk, messagebox, simpledialog
import queue

try:
    import requests
except Exception:
    requests = None

BASE = Path(__file__).parent
CITIES_FILE = BASE / "cities.json"
SAMPLE_FILE = BASE / "sample_weather.json"
CONFIG_FILE = BASE / "config.json"
CACHE_FILE = BASE / "cache.json"

# Ensure example files
if not CITIES_FILE.exists():
    CITIES_FILE.write_text(json.dumps(["Istanbul", "Ankara", "Izmir"], ensure_ascii=False, indent=2), encoding="utf-8")
if not SAMPLE_FILE.exists():
    SAMPLE_DATA = {
        "Istanbul": {"temp": 18, "desc": "Parçalı bulutlu", "humidity": 60},
        "Ankara": {"temp": 12, "desc": "Güneşli", "humidity": 45},
        "Izmir": {"temp": 22, "desc": "Az bulutlu", "humidity": 50}
    }
    SAMPLE_FILE.write_text(json.dumps(SAMPLE_DATA, ensure_ascii=False, indent=2), encoding="utf-8")
if not CONFIG_FILE.exists():
    CONFIG_FILE.write_text(json.dumps({"openweather_api_key": ""}, ensure_ascii=False, indent=2), encoding="utf-8")
if not CACHE_FILE.exists():
    CACHE_FILE.write_text(json.dumps({}, ensure_ascii=False), encoding="utf-8")

with CITIES_FILE.open(encoding="utf-8") as f:
    cities = json.load(f)
with SAMPLE_FILE.open(encoding="utf-8") as f:
    sample_data = json.load(f)
with CONFIG_FILE.open(encoding="utf-8") as f:
    config = json.load(f)
with CACHE_FILE.open(encoding="utf-8") as f:
    cache = json.load(f)

API_KEY = config.get("openweather_api_key", "").strip()

# Simple icon map by keyword
ICON_MAP = {
    "clear": "☀️", "cloud": "☁️", "rain": "🌧️", "storm": "⛈️",
    "snow": "❄️", "mist": "🌫️"
}

def save_cities():
    with CITIES_FILE.open("w", encoding="utf-8") as f:
        json.dump(cities, f, ensure_ascii=False, indent=2)

def save_cache():
    with CACHE_FILE.open("w", encoding="utf-8") as f:
        json.dump(cache, f, ensure_ascii=False, indent=2)

def detect_icon(desc):
    s = (desc or "").lower()
    for k, v in ICON_MAP.items():
        if k in s:
            return v
    return "🌤️"

def fetch_weather_api(city):
    if not API_KEY or requests is None:
        return None
    url = "https://api.openweathermap.org/data/2.5/weather"
    params = {"q": city, "appid": API_KEY, "units": "metric", "lang": "tr"}
    try:
        r = requests.get(url, params=params, timeout=7)
        r.raise_for_status()
        j = r.json()
        return {
            "temp": j.get("main", {}).get("temp"),
            "desc": j.get("weather", [{}])[0].get("description", ""),
            "humidity": j.get("main", {}).get("humidity"),
            "ts": int(time.time())
        }
    except Exception:
        return None

# Threaded worker for API to avoid UI blocking
worker_q = queue.Queue()

def worker_loop():
    while True:
        city, resp_q = worker_q.get()
        if city is None:
            break
        data = None
        # prefer API, fallback sample
        if API_KEY and requests is not None:
            data = fetch_weather_api(city)
        if not data:
            data = sample_data.get(city)
            if data:
                data = dict(data)
                data["ts"] = int(time.time())
        if data:
            cache[city] = data
            save_cache()
        resp_q.put(data)
        worker_q.task_done()

threading.Thread(target=worker_loop, daemon=True).start()

# GUI
root = tk.Tk()
root.title("Hava Durumu Paneli")
root.geometry("640x360")
style = ttk.Style(root)
style.theme_use("clam")

main = ttk.Frame(root, padding=10)
main.pack(fill="both", expand=True)

# Left: controls + list
left = ttk.Frame(main, width=220)
left.pack(side="left", fill="y")

ttk.Label(left, text="Şehirler", font=("Segoe UI", 11, "bold")).pack(anchor="w")
search_var = tk.StringVar()
search_entry = ttk.Entry(left, textvariable=search_var)
search_entry.pack(fill="x", pady=(4,6))
listbox = tk.Listbox(left, height=14)
listbox.pack(fill="both", expand=True)

def reload_list(filter_text=""):
    listbox.delete(0, "end")
    for c in cities:
        if filter_text.strip().lower() in c.lower():
            listbox.insert("end", c)
reload_list()

def add_city():
    name = simpledialog.askstring("Yeni şehir", "Şehir adı ekle:")
    if name:
        name = name.strip()
        if name and name not in cities:
            cities.append(name)
            save_cities()
            reload_list(search_var.get())

def remove_city():
    sel = listbox.curselection()
    if not sel: return
    city = listbox.get(sel[0])
    if messagebox.askyesno("Sil", f"{city} silinsin mi?"):
        cities.remove(city)
        save_cities()
        cache.pop(city, None)
        save_cache()
        reload_list(search_var.get())
        clear_info()

btn_frame = ttk.Frame(left)
btn_frame.pack(fill="x", pady=(6,0))
ttk.Button(btn_frame, text="Ekle", command=add_city).pack(side="left", fill="x", expand=True, padx=2)
ttk.Button(btn_frame, text="Sil", command=remove_city).pack(side="left", fill="x", expand=True, padx=2)

# Right: weather display
right = ttk.Frame(main, padding=(12,0))
right.pack(side="left", fill="both", expand=True)

title = ttk.Label(right, text="Hava Durumu", font=("Segoe UI", 13, "bold"))
title.pack(anchor="w")

card = ttk.Frame(right, relief="ridge", padding=12)
card.pack(fill="both", expand=True, pady=8)

icon_lbl = ttk.Label(card, text="🌤️", font=("Segoe UI Emoji", 48))
icon_lbl.grid(row=0, column=0, rowspan=3, padx=(0,12))

city_lbl = ttk.Label(card, text="-", font=("Segoe UI", 16, "bold"))
city_lbl.grid(row=0, column=1, sticky="w")
temp_lbl = ttk.Label(card, text="- °C", font=("Segoe UI", 20))
temp_lbl.grid(row=1, column=1, sticky="w")
desc_lbl = ttk.Label(card, text="-", font=("Segoe UI", 10), foreground="gray")
desc_lbl.grid(row=2, column=1, sticky="w")
hum_lbl = ttk.Label(card, text="Nem: -", font=("Segoe UI", 10))
hum_lbl.grid(row=3, column=1, sticky="w", pady=(8,0))
updated_lbl = ttk.Label(card, text="Güncelleme: -", font=("Segoe UI", 8), foreground="gray")
updated_lbl.grid(row=4, column=0, columnspan=2, sticky="w", pady=(8,0))

ctrl_frame = ttk.Frame(right)
ctrl_frame.pack(fill="x")
def refresh_selected():
    sel = listbox.curselection()
    if not sel:
        messagebox.showinfo("Bilgi", "Önce bir şehir seçin.")
        return
    city = listbox.get(sel[0])
    start_fetch(city)

ttk.Button(ctrl_frame, text="Yenile", command=refresh_selected).pack(side="left")
ttk.Button(ctrl_frame, text="API Anahtarı", command=lambda: edit_api_key()).pack(side="left", padx=6)
note = ttk.Label(right, text="Not: API anahtarı config.json içine yazılır. requests yoksa gerçek veri çekilemez.", foreground="gray")
note.pack(anchor="w", pady=(6,0))

def clear_info():
    city_lbl.config(text="-")
    temp_lbl.config(text="- °C")
    desc_lbl.config(text="-")
    hum_lbl.config(text="Nem: -")
    icon_lbl.config(text="🌤️")
    updated_lbl.config(text="Güncelleme: -")

# Async fetch orchestration
def start_fetch(city):
    city_lbl.config(text=city)
    temp_lbl.config(text="Yükleniyor...")
    desc_lbl.config(text="")
    hum_lbl.config(text="")
    icon_lbl.config(text="⏳")
    resp_q = queue.Queue()
    worker_q.put((city, resp_q))
    def waiter():
        data = resp_q.get()
        root.after(0, lambda: apply_weather(city, data))
    threading.Thread(target=waiter, daemon=True).start()

def apply_weather(city, info):
    if not info:
        messagebox.showinfo("Bilgi", f"{city} için veri alınamadı.")
        clear_info()
        return
    city_lbl.config(text=city)
    temp = info.get("temp")
    temp_lbl.config(text=f"{temp} °C" if temp is not None else "- °C")
    desc = info.get("desc", "")
    desc_lbl.config(text=desc or "-")
    hum = info.get("humidity")
    hum_lbl.config(text=f"Nem: %{hum}" if hum is not None else "Nem: -")
    icon_lbl.config(text=detect_icon(desc))
    ts = info.get("ts")
    if ts:
        updated_lbl.config(text="Güncelleme: " + time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(ts)))
    else:
        updated_lbl.config(text="Güncelleme: -")

def on_select(evt):
    sel = listbox.curselection()
    if not sel: return
    city = listbox.get(sel[0])
    # try cache first (valid 10 min)
    c = cache.get(city)
    if c and int(time.time()) - c.get("ts",0) < 600:
        apply_weather(city, c)
    else:
        start_fetch(city)

listbox.bind("<<ListboxSelect>>", on_select)
search_var.trace_add("write", lambda *a: reload_list(search_var.get()))

def edit_api_key():
    cur = config.get("openweather_api_key", "")
    new = simpledialog.askstring("API Anahtarı", "OpenWeatherMap API Key:", initialvalue=cur)
    if new is None:
        return
    config["openweather_api_key"] = new.strip()
    with CONFIG_FILE.open("w", encoding="utf-8") as f:
        json.dump(config, f, ensure_ascii=False, indent=2)
    global API_KEY
    API_KEY = config.get("openweather_api_key", "").strip()
    messagebox.showinfo("Bilgi", "API anahtarı kaydedildi. Yenile tuşuna basarak gerçek veriyi çekebilirsiniz.")

# Start with first city selected
if cities:
    listbox.select_set(0)
    root.after(200, lambda: on_select(None))

root.mainloop()
# ...existing code...
