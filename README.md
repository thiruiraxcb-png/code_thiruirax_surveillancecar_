
#include "esp_camera.h"
#include <WiFi.h>
#include <WebServer.h>

// Replace with your network credentials
const char* ssid = "YOUR_SSID";
const char* password = "YOUR_PASSWORD";

// Camera Model Definition (AI-Thinker)
#define CAMERA_MODEL_AI_THINKER
#include "camera_pins.h"

WebServer server(80);

// Motor Pins
const int IN1 = 12;
const int IN2 = 13;
const int IN3 = 15;
const int IN4 = 14;

// Embedded HTML UI with embedded stream and touch/click controls
const char PROGMEM INDEX_HTML[] = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>ESP32-CAM Robot Car</title>
  <style>
    body { font-family: Arial; text-align: background: #222; color: #fff; margin: 0; padding: 20px; background-color: #121212; color: #e0e0e0; }
    h2 { color: #00bcd4; }
    img { width: 100%; max-width: 400px; border-radius: 8px; border: 2px solid #00bcd4; }
    .btn { width: 90px; height: 90px; font-size: 24px; margin: 5px; background: #00bcd4; color: white; border: none; border-radius: 50%; cursor: pointer; outline: none; box-shadow: 0 4px #008ba3; }
    .btn:active { background: #008ba3; transform: translateY(4px); box-shadow: 0 0 #008ba3; }
    .control-pad { display: grid; grid-template-columns: repeat(3, 100px); justify-content: center; gap: 5px; margin-top: 20px; }
    .empty { visibility: hidden; }
  </style>
</head>
<body>
  <center>
    <h2>ESP32-CAM Surveillance Car</h2>
    <img src="" id="stream" crossorigin><br><br>
    <div class="control-pad">
      <div class="empty"></div>
      <button class="btn" ontouchstart="fetch('/action?go=forward')" onmousedown="fetch('/action?go=forward')">&#9650;</button>
      <div class="empty"></div>
      <button class="btn" ontouchstart="fetch('/action?go=left')" onmousedown="fetch('/action?go=left')">&#9668;</button>
      <button class="btn" ontouchstart="fetch('/action?go=stop')" onmousedown="fetch('/action?go=stop')">&#9632;</button>
      <button class="btn" ontouchstart="fetch('/action?go=right')" onmousedown="fetch('/action?go=right')">&#9658;</button>
      <div class="empty"></div>
      <button class="btn" ontouchstart="fetch('/action?go=backward')" onmousedown="fetch('/action?go=backward')">&#9660;</button>
      <div class="empty"></div>
    </div>
  </center>
  <script>
    window.onload = function() {
      document.getElementById("stream").src = window.location.origin + ":81/stream";
    }
  </script>
</body>
</html>
)rawliteral";

void handleRoot() {
  server.send(200, "text/html", INDEX_HTML);
}

void handleAction() {
  String state = server.arg("go");
  if (state == "forward") {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  } else if (state == "backward") {
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);
  } else if (state == "left") {
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  } else if (state == "right") {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);
  } else if (state == "stop") {
    digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);
  }
  server.send(200, "text/plain", "OK");
}

void startCameraServer();

void setup() {
  Serial.begin(115200);

  // Setup motor pins
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);

  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sccb_sda = SIOD_GPIO_NUM;
  config.pin_sccb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  if (psramFound()) {
    config.frame_size = FRAMESIZE_QVGA;
    config.jpeg_quality = 12;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_QQVGA;
    config.jpeg_quality = 12;
    config.fb_count = 1;
  }

  esp_camera_init(&config);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  // Start web server handlers
  server.on("/", HTTP_GET, handleRoot);
  server.on("/action", HTTP_GET, handleAction);
  server.begin();
  
  startCameraServer(); // Starts stream server on port 81

  Serial.println("");
  Serial.print("Open this URL in your phone browser: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
}
