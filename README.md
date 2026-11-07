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
    bytes_data = b''

    while True:
        # Read camera stream chunk by chunk
        bytes_data += stream.read(1024)
        a = bytes_data.find(b'\xff\xd8') # JPEG frame start boundary
        b = bytes_data.find(b'\xff\xd9') # JPEG frame end boundary

        if a != -1 and b != -1:
            jpg = bytes_data[a:b+2]
            bytes_data = bytes_data[b+2:]
            
            # Decode JPEG image array to OpenCV format
            frame = cv2.imdecode(np.frombuffer(jpg, dtype=np.uint8), cv2.IMREAD_COLOR)

            if frame is not None:
                # Add status overlay to frame
                cv2.putText(frame, "STATUS: LIVE STREAMING", (10, 30), 
                            cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
                
                # Display feed window
                cv2.imshow("Surveillance Car Feed", frame)

        # Handle keyboard input controls
        key = cv2.waitKey(1) & 0xFF
        if key == ord('w'):
            send_command("forward")
        elif key == ord('s'):
            send_command("backward")
        elif key == ord('a'):
            send_command("left")
        elif key == ord('d'):
            send_command("right")
        elif key == 32:  # Spacebar
            send_command("stop")
        elif key == ord('q'):
            print("Shutting down control system...")
            send_command("stop")
            break

    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
