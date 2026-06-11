#include "esp_camera.h"
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <WebServer.h>
#include <UniversalTelegramBot.h>

// ====== Wi-Fi (with STATIC IP) ======
const char* ssid = "Prathik";
const char* password = "12345678";
IPAddress local_IP(192,168,1,45);     // Static IP (change if needed)
IPAddress gateway(192,168,1,1);
IPAddress subnet(255,255,255,0);
IPAddress primaryDNS(8,8,8,8);

// ====== Telegram ======
#define BOTtoken "********************"
#define CHAT_ID "***************"

// ====== Camera Pins (AI Thinker) ======
#define PWDN_GPIO_NUM 32
#define RESET_GPIO_NUM -1
#define XCLK_GPIO_NUM 0
#define SIOD_GPIO_NUM 26
#define SIOC_GPIO_NUM 27
#define Y9_GPIO_NUM 35
#define Y8_GPIO_NUM 34
#define Y7_GPIO_NUM 39
#define Y6_GPIO_NUM 36
#define Y5_GPIO_NUM 21
#define Y4_GPIO_NUM 19
#define Y3_GPIO_NUM 18
#define Y2_GPIO_NUM 5
#define VSYNC_GPIO_NUM 25
#define HREF_GPIO_NUM 23
#define PCLK_GPIO_NUM 22

#define PIR_PIN 13
#define FENCE_PIN 14
#define FLASH_LED_PIN 4
#define BUZZER_PIN 2

WiFiClientSecure client;
UniversalTelegramBot bot(BOTtoken, client);
WebServer server(80);

bool surveillanceEnabled = true;
unsigned long lastMotion = 0;
unsigned long lastCheck = 0;
const unsigned long botDelay = 3000;

// ====== Capture and Send to Telegram ======
void sendPhotoToTelegram() {
  digitalWrite(FLASH_LED_PIN, HIGH);
  delay(400);
  camera_fb_t *fb = esp_camera_fb_get();
  digitalWrite(FLASH_LED_PIN, LOW);
  if (!fb) return;

  String head = "--boundary\r\nContent-Disposition: form-data; name=\"chat_id\"\r\n\r\n" + String(CHAT_ID) +
                "\r\n--boundary\r\nContent-Disposition: form-data; name=\"photo\"; filename=\"photo.jpg\"\r\n"
                "Content-Type: image/jpeg\r\n\r\n";
  String tail = "\r\n--boundary--\r\n";

  if (client.connect("api.telegram.org", 443)) {
    client.println("POST /bot" + String(BOTtoken) + "/sendPhoto HTTP/1.1");
    client.println("Host: api.telegram.org");
    client.println("Content-Type: multipart/form-data; boundary=boundary");
    client.println("Content-Length: " + String(head.length() + fb->len + tail.length()));
    client.println();
    client.print(head);
    uint8_t *buf = fb->buf;
    for (size_t n = 0; n < fb->len; n += 512)
      client.write(buf + n, (fb->len - n) < 512 ? (fb->len - n) : 512);
    client.print(tail);
  }
  esp_camera_fb_return(fb);
}

// ====== Motion Detection ======
void checkMotion() {
  int pir = digitalRead(PIR_PIN);
  int fence = digitalRead(FENCE_PIN);
  if (pir == HIGH && fence == HIGH && surveillanceEnabled && millis() - lastMotion > 15000) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(200);
    digitalWrite(BUZZER_PIN, LOW);
    bot.sendMessage(CHAT_ID, "🚨 Motion detected! Capturing image...");
    sendPhotoToTelegram();
    lastMotion = millis();
  }
}

// ====== Telegram Commands ======
void handleNewMessages(int n) {
  for (int i = 0; i < n; i++) {
    String text = bot.messages[i].text;
    if (text == "/start") {
      bot.sendMessageWithReplyKeyboard(CHAT_ID,
        "🤖 *AgriSafeX Smart Farm*\nCommands:\n📸 /photo\n🌱 /soil\n📊 /status",
        "Markdown", "[[\"📸 /photo\"],[\"🌱 /soil\"],[\"📊 /status\"]]", true);
    }
    if (text == "/photo" || text == "📸 /photo") {
      bot.sendMessage(CHAT_ID, "📷 Capturing...");
      sendPhotoToTelegram();
    }
    if (text == "/soil" || text == "🌱 /soil") {
      bot.sendMessage(CHAT_ID, "🌱 Checking soil moisture...");
      Serial.println("GET_SOIL");
    }
    if (text == "/status" || text == "📊 /status") {
      String msg = "📊 Farm Status:\n";
      msg += surveillanceEnabled ? "🔒 Surveillance: ON\n" : "🔓 Surveillance: OFF\n";
      bot.sendMessage(CHAT_ID, msg, "Markdown");
    }
  }
}

// ====== Serial from Nano ======
void checkSerialFromNano() {
  if (Serial.available()) {
    String data = Serial.readStringUntil('\n');
    data.trim();
    if (data.startsWith("SOIL=")) {
      String m = data.substring(5);
      bot.sendMessage(CHAT_ID, "🌱 Soil Moisture: " + m + "%", "");
    } else if (data == "FARMER_ENTER") {
      surveillanceEnabled = false;
      bot.sendMessage(CHAT_ID, "👨‍🌾 Farmer Entered — Surveillance OFF");
    } else if (data == "FARMER_EXIT") {
      surveillanceEnabled = true;
      bot.sendMessage(CHAT_ID, "🚪 Farmer Exited — Surveillance ON");
    }
  }
}

// ====== Local Web Server ======
void startWebServer() {
  server.on("/", HTTP_GET, []() {
    String html = "<html><body><h2>🌾 AgriSafeX ESP32-CAM</h2>";
    html += "<p>📸 <a href='/stream'>Live Stream</a></p>";
    html += "<p>🔒 Mode: " + String(surveillanceEnabled ? "Surveillance ON" : "OFF") + "</p>";
    html += "</body></html>";
    server.send(200, "text/html", html);
  });

  server.on("/stream", HTTP_GET, []() {
    server.sendHeader("Location", "http://" + WiFi.localIP().toString() + ":81/stream");
    server.send(302, "text/plain", "");
  });

  server.begin();
  Serial.println("✅ Web Server started on port 80");
}

// ====== SETUP ======
void setup() {
  Serial.begin(9600);
  pinMode(PIR_PIN, INPUT);
  pinMode(FENCE_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(FLASH_LED_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(FLASH_LED_PIN, LOW);

  WiFi.config(local_IP, gateway, subnet, primaryDNS);
  WiFi.begin(ssid, password);
  Serial.print("📶 Connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\n✅ Wi-Fi connected!");
  Serial.print("🌐 Access your camera at: http://");
  Serial.println(WiFi.localIP());

  client.setInsecure();

  camera_config_t c;
  c.ledc_channel = LEDC_CHANNEL_0;
  c.ledc_timer = LEDC_TIMER_0;
  c.pin_d0 = Y2_GPIO_NUM; c.pin_d1 = Y3_GPIO_NUM; c.pin_d2 = Y4_GPIO_NUM; c.pin_d3 = Y5_GPIO_NUM;
  c.pin_d4 = Y6_GPIO_NUM; c.pin_d5 = Y7_GPIO_NUM; c.pin_d6 = Y8_GPIO_NUM; c.pin_d7 = Y9_GPIO_NUM;
  c.pin_xclk = XCLK_GPIO_NUM; c.pin_pclk = PCLK_GPIO_NUM; c.pin_vsync = VSYNC_GPIO_NUM;
  c.pin_href = HREF_GPIO_NUM; c.pin_sscb_sda = SIOD_GPIO_NUM; c.pin_sscb_scl = SIOC_GPIO_NUM;
  c.pin_pwdn = PWDN_GPIO_NUM; c.pin_reset = RESET_GPIO_NUM;
  c.xclk_freq_hz = 20000000; c.pixel_format = PIXFORMAT_JPEG;
  c.frame_size = FRAMESIZE_VGA; c.jpeg_quality = 10; c.fb_count = 1;

  esp_camera_init(&c);
  startWebServer();
  bot.sendMessage(CHAT_ID, "🤖 AgriSafeX Online!\n🌐 Access: http://" + WiFi.localIP().toString());
}

// ====== LOOP ======
void loop() {
  server.handleClient();
  if (millis() - lastCheck > botDelay) {
    int n = bot.getUpdates(bot.last_message_received + 1);
    if (n) handleNewMessages(n);
    lastCheck = millis();
  }
  checkSerialFromNano();
  checkMotion();
}
