import collections
from datetime import datetime
import threading
import time
import tkinter as tk
from tkinter import ttk
import csv
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from matplotlib.figure import Figure
import pandas as pd

# lab8
import serial

buffer = collections.deque(maxlen=200)


def acquire():
  try:
    ser = serial.Serial("/dev/ttyACM0", 9600, timeout=1)
    time.sleep(2)
    while True:
      line = ser.readline().decode("utf-8", "ignore").strip()
      if line:
        parts = line.split(",")
        if len(parts) >= 3:
          buffer.append(
              (time.time(), float(parts[0]), float(parts[1]), int(parts[2]))
          )
  except Exception as e:
    print("Thread serial error:", e)


threading.Thread(target=acquire, daemon=True).start()

root = tk.Tk()
root.title("Lab7 - Sensor Dashboard")
root.geometry("600x600")

table_frame = ttk.Frame(root)
table_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

dashboard_column = ("timestamp", "temperature", "humidity", "motion")
tree = ttk.Treeview(table_frame, columns=dashboard_column, show="headings")

tree.heading("timestamp", text="Timestamp")
tree.heading("temperature", text="Temperature")
tree.heading("humidity", text="humidity")
tree.heading("motion", text="Motion")

tree.column("timestamp", width=180, anchor="center")
tree.column("temperature", width=120, anchor="center")
tree.column("humidity", width=120, anchor="center")
tree.column("motion", width=80, anchor="center")

scrollbar = ttk.Scrollbar(table_frame, orient=tk.VERTICAL, command=tree.yview)
tree.configure(yscrollcommand=scrollbar.set)

tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

chart_frame = ttk.Frame(root)
chart_frame.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=10, pady=10)
fig = Figure(figsize=(5, 4), dpi=100)
ax = fig.add_subplot(111)
canvas = FigureCanvasTkAgg(fig, master=chart_frame)
canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True)


def refresh():

  # lab8
  if len(buffer) > 0:
    for item in tree.get_children():
      tree.delete(item)
    temperature = []
    humidities = []
    motions = []
    for item in list(buffer):
      t_num, temp, hum, mot = item
      ts_str = datetime.fromtimestamp(t_num).strftime("%Y-%m-%d %H:%M:%S")

      temperature.append(temp)
      humidities.append(hum)
      motions.append(mot)
      tree.insert("", 0, values=(ts_str, temp, hum, mot))
    children = tree.get_children()
    if len(children) > 30:
      for child in children[30:]:
        tree.delete(child)

    ax.clear()
    ax.plot(
        temperature[-30:],
        label="Temp (C)",
        color="red",
        marker="o",
        linewidth=1.5,
    )
    ax.plot(
        humidities[-30:],
        label="Humidity (%)",
        color="orange",
        marker="o",
        linewidth=1.5,
    )
    ax.plot(
        motions[-30:],
        label="Motion",
        color="blue",
        marker="o",
        linewidth=1.5,
    )

    ax.set_title("Live Hardware Sensor (Lab8)")
    ax.set_xticks([])
    ax.set_xticklabels([])
    ax.legend(loc="upper left", fontsize=8)
    canvas.draw()

  root.after(1000, refresh)


root.after(1000, refresh)
root.mainloop()
