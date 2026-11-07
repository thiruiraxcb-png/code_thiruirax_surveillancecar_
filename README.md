import io
import logging
import socketserver
from http import server
from threading import Condition
from gpiozero import Motor
from Picamera2 import Picamera2

# --- Motor Setup ---
left_motor = Motor(forward=17, backward=27)
right_motor = Motor(forward=22, backward=23)

def car_stop():
    left_motor.stop()
    right_motor.stop()

def car_forward():
    left_motor.forward()
    right_motor.forward()

def car_backward():
    left_motor.backward()
    right_motor.backward()

def car_left():
    left_motor.backward()
    right_motor.forward()

def car_right():
    left_motor.forward()
    right_motor.backward()

# --- Streaming Output Setup ---
class StreamingOutput(object):
    def __init__(self):
        self.frame = None
        self.buffer = io.BytesIO()
        self.condition = Condition()

    def write(self, buf):
        if buf.startswith(b'\xff\xd8'):
            self.buffer.truncate()
            with self.condition:
                self.frame = self.buffer.getvalue()
                self.condition.notify_all()
            self.buffer.seek(0)
        return self.buffer.write(buf)

# --- Web Server Handler ---
PAGE = """\
<html>
<head>
<title>Surveillance Car</title>
<style>
  body { text-align: center; background-color: #1a1a1a; color: white; font-family: Arial; }
  .btn { width: 80px; height: 50px; font-size: 18px; margin: 5px; cursor: pointer; }
</style>
</head>
<body>
<h1>Surveillance Car Control</h1>
<img src="stream.mjpg" width="640" height="480" /><br/><br/>
<div>
  <button class="btn" onclick="fetch('/forward')">▲</button><br/>
  <button class="btn" onclick="fetch('/left')">◄</button>
  <button class="btn" onclick="fetch('/stop')">■</button>
  <button class="btn" onclick="fetch('/right')">►</button><br/>
  <button class="btn" onclick="fetch('/backward')">▼</button>
</div>
</body>
</html>
"""

class StreamingHandler(server.BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/':
            self.send_response(301)
            self.send_header('Location', '/index.html')
            self.end_headers()
        elif self.path == '/index.html':
            content = PAGE.encode('utf-8')
            self.send_response(200)
            self.send_header('Content-Type', 'text/html')
            self.send_header('Content-Length', len(content))
            self.end_headers()
            self.wfile.write(content)
        elif self.path == '/forward':
            car_forward()
            self.send_response(200)
            self.end_headers()
        elif self.path == '/backward':
            car_backward()
            self.send_response(200)
            self.end_headers()
        elif self.path == '/left':
            car_left()
            self.send_response(200)
            self.end_headers()
        elif self.path == '/right':
            car_right()
            self.send_response(200)
            self.end_headers()
        elif self.path == '/stop':
            car_stop()
            self.send_response(200)
            self.end_headers()
        elif self.path == '/stream.mjpg':
            self.send_response(200)
            self.send_header('Age', 0)
            self.send_header('Cache-Control', 'no-cache, private')
            self.send_header('Pragma', 'no-cache')
            self.send_header('Content-Type', 'multipart/x-mixed-replace; boundary=FRAME')
            self.end_headers()
            try:
                while True:
                    with output.condition:
                        output.condition.wait()
                        frame = output.frame
                    self.wfile.write(b'--FRAME\r\n')
                    self.send_header('Content-Type', 'image/jpeg')
                    self.send_header('Content-Length', len(frame))
                    self.end_headers()
                    self.wfile.write(frame)
                    self.wfile.write(b'\r\n')
            except Exception as e:
                logging.warning(f'Removed streaming client {self.client_address}: {e}')
        else:
            self.send_error(404)
            self.end_headers()

class StreamingServer(socketserver.ThreadingMixIn, server.HTTPServer):
    allow_reuse_address = True
    daemon_threads = True

# --- Main Execution ---
output = StreamingOutput()
picam2 = Picamera2()
picam2.configure(picam2.create_video_configuration({"size": (640, 480)}))
picam2.start_recording(output, format='jpeg')

try:
    address = ('', 8000)
    server = StreamingServer(address, StreamingHandler)
    print("Server starting at http://<PI_IP_ADDRESS>:8000")
    server.serve_forever()
finally:
    picam2.stop_recording()
    car_stop()
