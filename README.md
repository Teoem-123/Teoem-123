#include "esp_camera.h"
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "YOUR_SSID"; // Thay bằng SSID của bạn
const char* password = "YOUR_PASSWORD"; // Thay bằng mật khẩu của bạn

// Định nghĩa chân cho động cơ
#define MOTOR1_IN1 13
#define MOTOR1_IN2 12
#define MOTOR2_IN1 14
#define MOTOR2_IN2 27

WebServer server(80);

void setup() {
  Serial.begin(115200);

  // Kết nối WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");

  // Khởi tạo camera
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = 32;
  config.pin_d1 = 33;
  config.pin_d2 = 34;
  config.pin_d3 = 35;
  config.pin_d4 = 36;
  config.pin_d5 = 39;
  config.pin_d6 = 26;
  config.pin_d7 = 25;
  config.pin_xclk = 0;
  config.pin_pclk = 22;
  config.pin_vsync = 25;
  config.pin_href = 23;
  config.pin_sscb_sda = 21;
  config.pin_sscb_scl = 19;
  config.pin_pwdn = -1;
  config.pin_reset = -1;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  if (psramFound()) {
    config.frame_size = FRAMESIZE_SVGA;
    config.jpeg_quality = 12;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_VGA;
    config.jpeg_quality = 12;
    config.fb_count = 1;
  }

  esp_camera_init(&config);

  // Cấu hình chân động cơ
  pinMode(MOTOR1_IN1, OUTPUT);
  pinMode(MOTOR1_IN2, OUTPUT);
  pinMode(MOTOR2_IN1, OUTPUT);
  pinMode(MOTOR2_IN2, OUTPUT);

  // Cấu hình server
  server.on("/", HTTP_GET, handleRoot);
  server.on("/camera", HTTP_GET, handleCamera);
  server.on("/move", HTTP_GET, handleMove);
  server.begin();
}

void loop() {
  server.handleClient();
}

void handleRoot() {
  String html = "<html><body>";
  html += "<h1>Xe Điều Khiển</h1>";
  html += "<img id='camera' src='/camera' width='320' height='240'>";
  html += "<br>";
  html += "<button onclick=\"move('forward')\">Tiến</button>";
  html += "<button onclick=\"move('backward')\">Lùi</button>";
  html += "<button onclick=\"move('left')\">Trái</button>";
  html += "<button onclick=\"move('right')\">Phải</button>";
  html += "<button onclick=\"move('stop')\">Dừng</button>";
  html += "<script>function move(direction) { fetch('/move?dir=' + direction); } setInterval(() => { document.getElementById('camera').src = '/camera?' + new Date().getTime(); }, 100);</script>";
  html += "</body></html>";
  server.send(200, "text/html", html);
}

void handleCamera() {
  camera_fb_t *fb = esp_camera_fb_get();
  if (!fb) {
    Serial.println("Camera capture failed");
    server.send(500);
    return;
  }
  // Chuyển đổi buffer thành kiểu dữ liệu phù hợp
  server.send(fb->len, "image/jpeg", (const char *)fb->buf);
  esp_camera_fb_return(fb);
}


void handleMove() {
  if (server.arg("dir") == "forward") {
    moveForward();
  } else if (server.arg("dir") == "backward") {
    moveBackward();
  } else if (server.arg("dir") == "left") {
    turnLeft();
  } else if (server.arg("dir") == "right") {
    turnRight();
  } else if (server.arg("dir") == "stop") {
    stopMotors();
  }
  server.send(200, "text/plain", "OK");
}

void moveForward() {
  digitalWrite(MOTOR1_IN1, HIGH);
  digitalWrite(MOTOR1_IN2, LOW);
  digitalWrite(MOTOR2_IN1, HIGH);
  digitalWrite(MOTOR2_IN2, LOW);
}

void moveBackward() {
  digitalWrite(MOTOR1_IN1, LOW);
  digitalWrite(MOTOR1_IN2, HIGH);
  digitalWrite(MOTOR2_IN1, LOW);
  digitalWrite(MOTOR2_IN2, HIGH);
}

void turnLeft() {
  digitalWrite(MOTOR1_IN1, LOW);
  digitalWrite(MOTOR1_IN2, HIGH);
  digitalWrite(MOTOR2_IN1, HIGH);
  digitalWrite(MOTOR2_IN2, LOW);
}

void turnRight() {
  digitalWrite(MOTOR1_IN1, HIGH);
  digitalWrite(MOTOR1_IN2, LOW);
  digitalWrite(MOTOR2_IN1, LOW);
  digitalWrite(MOTOR2_IN2, HIGH);
}

void stopMotors() {
  digitalWrite(MOTOR1_IN1, LOW);
  digitalWrite(MOTOR1_IN2, LOW);
  digitalWrite(MOTOR2_IN1, LOW);
  digitalWrite(MOTOR2_IN2, LOW);
}

