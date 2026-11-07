 

import cv2
import numpy as np
import urllib.request
import socket

# --- Configuration ---
# Replace with the IP address printed in your ESP32-CAM Serial Monitor
ESP32_IP = "http://192.168.1.100"  
STREAM_URL = f"{ESP32_IP}:81/stream"  # Default stream port for ESP32-CAM

def send_command(command):
    """Send movement commands to the surveillance car server."""
    try:
        url = f"{ESP32_IP}/control?var=car&val={command}"
        urllib.request.urlopen(url, timeout=1)
        print(f"Command Sent: {command}")
    except Exception as e:
        print(f"Failed to send command '{command}': {e}")

def main():
    print("=" * 50)
    print(" Surveillance Car Remote Control System ")
    print(" Controls:")
    print("   [W] - Forward | [S] - Backward")
    print("   [A] - Left    | [D] - Right")
    print("   [Space] - Stop")
    print("   [Q] - Quit Application")
    print("=" * 50)

    # Open video stream from ESP32-CAM
    stream = urllib.request.urlopen(STREAM_URL)
    bytes_data