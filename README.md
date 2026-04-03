# Automacion-Pintura-PID-Vision
Sistema automatizado de pintura y clasificación con control PID y visión artificial (Simulación PLC en Python)
+SIMULADOR DE PLC+
import tkinter as tk
import random
import threading
import time
from datetime import datetime
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from tkinter import messagebox

# VARIABLES PLC
# =============================
SP = 5.0
PV = 0.0

state = "STOP"
emergency = False

LINE_SPEED = 5.0

motor = "OK"
vision = "OK"
belt = "OK"
safety = "OK"

pv_hist = []
sp_hist = []

history = []
fault_active = False

# CAUSAS INDUSTRIALES
# =============================
def cause(system, status):
    if system == "Motor":
        return "Fricción elevada o suciedad en motor"
    if system == "Vision":
        return "Lente sucia o cámara desalineada"
    if system == "Belt":
        return "Atasco en banda transportadora"
    if system == "Safety":
        return "Paro de emergencia activado"
    return "Normal"

# LOG FALLAS
# =============================
def log_fault(system, status):
    if status != "OK":
        history.append({
            "time": datetime.now().strftime("%H:%M:%S"),
            "system": system,
            "status": status,
            "cause": cause(system, status)
        })

# RESET PLC
# =============================
def reset_plc():
    global SP, PV, state, emergency
    global motor, vision, belt, safety
    global pv_hist, sp_hist, history, fault_active

    SP = 5.0
    PV = 0.0
    state = "STOP"
    emergency = False

    motor = "OK"
    vision = "OK"
    belt = "OK"
    safety = "OK"

    pv_hist.clear()
    sp_hist.clear()
    history.clear()
    fault_active = False

# FALLAS ALEATORIAS
# =============================
def faults():
    global motor, vision, belt, safety, fault_active

    if random.random() < 0.02:
        motor = random.choice(["OK", "WARNING", "FAULT"])
        log_fault("Motor", motor)

    if random.random() < 0.02:
        vision = random.choice(["OK", "WARNING", "FAULT"])
        log_fault("Vision", vision)

    if random.random() < 0.015:
        belt = random.choice(["OK", "FAULT"])
        log_fault("Belt", belt)

    if random.random() < 0.01:
        safety = random.choice(["OK", "FAULT"])
        log_fault("Safety", safety)

    fault_active = ("FAULT" in [motor, vision, belt, safety])

# PLANTA
# =============================
def plant():
    global PV

    if state != "RUN" or emergency:
        return

    PV += random.uniform(-0.2, 0.3) * (LINE_SPEED / 5)
    PV = max(0, min(10, PV))

# LOOP PLC (THREAD)
# =============================
def loop():
    while True:
        try:
            if state == "RUN" and not emergency:
                plant()
                faults()

                pv_hist.append(PV)
                sp_hist.append(SP)

                if len(pv_hist) > 120:
                    pv_hist.pop(0)
                    sp_hist.pop(0)

            update_ui()
            time.sleep(0.1)

        except:
            pass

# UI
# =============================
root = tk.Tk()
root.title("SCADA PLC - SISTEMA INDUSTRIAL COMPLETO")
root.geometry("1400x800")
root.configure(bg="#1e1e1e")

main = tk.Frame(root, bg="#1e1e1e")
main.pack(fill="both", expand=True)

left = tk.Frame(main, bg="black", width=500)
center = tk.Frame(main, bg="#2b2b2b", width=350)
right = tk.Frame(main, bg="#111111", width=500)

left.pack(side="left", fill="both", expand=True)
center.pack(side="left", fill="y")
right.pack(side="right", fill="both", expand=True)

# BANDA
# =============================
canvas = tk.Canvas(left, bg="black", height=350)
canvas.pack(fill="both", expand=True)

# PLC PANEL
# =============================
tk.Label(center, text="PLC CONTROL",
         fg="white", bg="#2b2b2b",
         font=("Arial", 18, "bold")).pack()

state_label = tk.Label(center, text="STATE: STOP",
                       fg="red", bg="#2b2b2b",
                       font=("Arial", 14))
state_label.pack()

sp_entry = tk.Entry(center)
sp_entry.insert(0, "5")
sp_entry.pack()

def set_sp():
    global SP
    try:
        SP = float(sp_entry.get())
    except:
        pass

tk.Button(center, text="SET SP", command=set_sp, bg="blue").pack()

speed_entry = tk.Entry(center)
speed_entry.insert(0, "5")
speed_entry.pack()

def set_speed():
    global LINE_SPEED
    try:
        LINE_SPEED = float(speed_entry.get())
    except:
        pass

tk.Button(center, text="SET SPEED", command=set_speed, bg="cyan").pack()

def start():
    global state
    state = "RUN"

def stop():
    global state
    state = "STOP"

def emergency_stop():
    global emergency
    emergency = True

def reset_emergency():
    global emergency
    emergency = False

tk.Button(center, text="START", bg="green", fg="white", command=start).pack()
tk.Button(center, text="STOP", bg="orange", fg="white", command=stop).pack()
tk.Button(center, text="EMERGENCY STOP", bg="red", fg="white", command=emergency_stop).pack()
tk.Button(center, text="RESET EMERGENCY", command=reset_emergency).pack()
tk.Button(center, text="RESET PLC", bg="purple", fg="white", command=reset_plc).pack(pady=10)

# RIGHT PANEL (ALARM LOG)
# =============================
tk.Label(right, text="ALARM LOG",
         fg="white", bg="#111111",
         font=("Arial", 16, "bold")).pack()

history_box = tk.Listbox(right, height=20, width=60)
history_box.pack()

def show_info(event):
    if not history:
        return
    idx = history_box.curselection()
    if idx:
        item = history[-(idx[0]+1)]
        messagebox.showinfo("DIAGNOSTIC",
            f"System: {item['system']}\n"
            f"Status: {item['status']}\n"
            f"Cause: {item['cause']}")

history_box.bind("<Double-Button-1>", show_info)

# GRAFICA
# =============================
fig, ax = plt.subplots()
graph = FigureCanvasTkAgg(fig, master=root)
graph.get_tk_widget().pack(fill="both", expand=True)

#  UPDATE UI
# =============================
def update_ui():
    state_label.config(text=f"STATE: {state}",
                       fg="green" if state=="RUN" else "red")

    canvas.delete("all")

    if emergency:
        canvas.create_text(300, 150,
            text="EMERGENCY STOP ACTIVE",
            fill="red",
            font=("Arial", 20, "bold"))
    else:
        canvas.create_rectangle(50, 150, 900, 250, fill="#444")

        for i in range(len(pv_hist)):
            x = 50 + i * (LINE_SPEED * 1.2)
            canvas.create_rectangle(x, 170, x+10, 230, fill="yellow")

    history_box.delete(0, tk.END)
    for h in history[-10:]:
        history_box.insert(0, f"{h['time']} | {h['system']} | {h['status']}")

    ax.clear()
    ax.plot(pv_hist, label="PV")
    ax.plot(sp_hist, label="SP")
    ax.legend()
    graph.draw()

#  START THREAD
# =============================
threading.Thread(target=loop, daemon=True).start()

root.mainloop()




SISTEMA DE VISION 
import cv2
import os
import tkinter as tk
from tkinter import Label, Button
from PIL import Image, ImageTk

# Carpeta de imágenes
carpeta = "imagenes"
imagenes = os.listdir(carpeta)
index = 0

# ----------------- FUNCIÓN DE ANÁLISIS -----------------
def analizar_imagen(path):
    img = cv2.imread(path)

    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    _, thresh = cv2.threshold(gray, 100, 400, cv2.THRESH_BINARY)

    white = cv2.countNonZero(thresh)
    total = thresh.size

    quality = (white / total) * 300

    if quality >= 95:
        estado = "OK"
        color = "green"
    else:
        estado = "NOK"
        color = "red"

    return img, quality, estado, color

# ----------------- INTERFAZ -----------------
ventana = tk.Tk()
ventana.title("Cámara Leonardo SJ - Sistema de Visión Artificial")
ventana.geometry("850x650")
ventana.configure(bg="black")

titulo = Label(
    ventana,
    text="SISTEMA DE VISIÓN ARTIFICIAL",
    font=("Arial", 20, "bold"),
    fg="white",
    bg="black"
)
titulo.pack(pady=10)

estado_label = Label(
    ventana,
    text="",
    font=("Arial", 18),
    bg="black"
)
estado_label.pack(pady=10)

imagen_label = Label(ventana)
imagen_label.pack()

# ----------------- MOSTRAR IMAGEN -----------------
def mostrar():
    global index

    if index >= len(imagenes):
        index = 0

    path = os.path.join(carpeta, imagenes[index])

    img, quality, estado, color = analizar_imagen(path)

    img_rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    img_pil = Image.fromarray(img_rgb)
    img_pil = img_pil.resize((450, 300))

    img_tk = ImageTk.PhotoImage(img_pil)

    imagen_label.config(image=img_tk)
    imagen_label.image = img_tk

    estado_label.config(
        text=f"Calidad: {quality:.2f}%  |  Estado: {estado}",
        fg=color
    )

    index += 1

# ----------------- BOTÓN -----------------
btn = Button(
    ventana,
    text="SIGUIENTE PIEZA",
    command=mostrar,
    font=("Arial", 14, "bold"),
    bg="gray",
    fg="white"
)
btn.pack(pady=20)

ventana.mainloop()

FABRICA DE CLASIFICACION DE PIEZAS 
import cv2
import os
import numpy as np
import time
# -----------------------------
# RUTA
# -----------------------------
carpeta = r"C:\Users\carlo\Desktop\Vision_Artificial\imagenes"
imagenes = sorted([f for f in os.listdir(carpeta) if f.endswith(".jpg")])

# -----------------------------
# VARIABLES
# -----------------------------
contador_ok = 0
contador_nok = 0
velocidad = 5
running = False

piezas_activas = []
indice_imagen = 0

tiempo_ultima_pieza = 0
intervalo_piezas = 1.5  # segundos entre piezas
# -----------------------------
# DETECCIÓN POR COLOR
# -----------------------------
def clasificar_color(img):
    hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)

    lower_red = np.array([0, 120, 70])
    upper_red = np.array([10, 255, 255])

    mask = cv2.inRange(hsv, lower_red, upper_red)
    red_pixels = cv2.countNonZero(mask)

    return "NOK" if red_pixels > 500 else "OK"


# -----------------------------
# CREAR PIEZA
# -----------------------------
def crear_pieza():
    global indice_imagen

    if indice_imagen >= len(imagenes):
        return

    ruta = os.path.join(carpeta, imagenes[indice_imagen])
    img = cv2.imread(ruta)

    if img is None:
        indice_imagen += 1
        return

    img = cv2.resize(img, (80, 80))
    resultado = clasificar_color(img)

    pieza = {
        "img": img,
        "x": 0,
        "y": 260,
        "resultado": resultado
    }

    piezas_activas.append(pieza)
    indice_imagen += 1


# -----------------------------
# BUCLE PRINCIPAL
# -----------------------------
while True:

    frame = 255 * np.ones((500, 900, 3), dtype="uint8")

    # SCADA
    cv2.putText(frame, "SCADA - LINEA AUTOMATIZADA", (20, 30),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,0,0), 2)

    cv2.putText(frame, f"OK: {contador_ok}", (20, 70),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,255,0), 2)

    cv2.putText(frame, f"NOK: {contador_nok}", (20, 100),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,0,255), 2)

    estado = "RUNNING" if running else "STOPPED"
    color_estado = (0,255,0) if running else (0,0,255)

    cv2.putText(frame, f"Estado: {estado}", (20, 140),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, color_estado, 2)

    cv2.putText(frame, f"Velocidad: {velocidad}", (20, 180),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,0,0), 2)

    # CINTA
    cv2.rectangle(frame, (0, 250), (900, 350), (50, 50, 50), -1)

    # ZONAS
    cv2.rectangle(frame, (700, 200), (880, 400), (0,255,0), 2)
    cv2.putText(frame, "EMPAQUETADO", (700, 190),
                cv2.FONT_HERSHEY_SIMPLEX, 0.6, (0,255,0), 2)

    cv2.rectangle(frame, (400, 350), (600, 480), (0,0,255), 2)
    cv2.putText(frame, "RECHAZO", (420, 490),
                cv2.FONT_HERSHEY_SIMPLEX, 0.6, (0,0,255), 2)

    # GENERAR PIEZAS
    if running:
         ahora = time.time()

         if (ahora - tiempo_ultima_pieza) > intervalo_piezas:
             if len(piezas_activas) < 0.2:
                 crear_pieza()
                 tiempo_ultima_pieza = ahora

    # MOVER PIEZAS
    nuevas_piezas = []

    for pieza in piezas_activas:
        x, y = pieza["x"], pieza["y"]
        img = pieza["img"]
        resultado = pieza["resultado"]

        # Movimiento
        if x < 600:
            x += velocidad
        else:
            if resultado == "OK":
                x += velocidad
            else:
                y += velocidad

        # 🔧 FIX ERROR BORDES
        h, w, _ = frame.shape
        if 0 <= x < w-80 and 0 <= y < h-80:
            frame[y:y+80, x:x+80] = img

        # Salida
        if x > 850:
            contador_ok += 1
        elif y > 450:
            contador_nok += 1
        else:
            pieza["x"], pieza["y"] = x, y
            nuevas_piezas.append(pieza)

    piezas_activas = nuevas_piezas

    # MOSTRAR
    cv2.imshow("Fabrica SCADA Ultra PRO", frame)

    # CONTROLES
    key = cv2.waitKey(30) & 0xFF

    if key == ord('q'):
        break
    elif key == ord('s'):
        running = True
    elif key == ord('p'):
        running = False
    elif key == ord('+'):
        velocidad += 1
    elif key == ord('-'):
        velocidad = max(1, velocidad - 1)

cv2.destroyAllWindows()
