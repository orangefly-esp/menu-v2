#include <Wire.h> 
#include <Adafruit_GFX.h> 
#include <Adafruit_SSD1306.h> 
#include <WiFi.h> 
#include <WebServer.h> 
#include <esp_wifi.h> 
#include <vector> 
#include <ESPmDNS.h> 
#include <DNSServer.h> 
#include <Preferences.h> 
#include <time.h> 
#include <IRremoteESP8266.h> 
#include <IRsend.h> 

// ========================================================================= 
//                         CẤU HÌNH & OLED 
// ========================================================================= 
#define SCREEN_WIDTH 128 
#define SCREEN_HEIGHT 64 
#define OLED_RESET    -1 
#define SCREEN_ADDRESS 0x3C 
#define I2C_SDA 20 
#define I2C_SCL 21 

// Định nghĩa 5 nút 
#define BTN_LEFT   2 
#define BTN_RIGHT  3 // Dùng làm SPACES 
#define BTN_SELECT 4 // Nút Giữ 
#define BTN_DOWN   5 
#define BTN_UP     7 // Nút UP 

// Chọn LED tích hợp (GPIO8 trên ESP32-C3 SuperMini - Active LOW) 
const int LED_PIN = 8; 
#define IR_LED_PIN 6 
IRsend irsend(IR_LED_PIN); 

// Cấu hình NTP cho Múi giờ Việt Nam (GMT+7) 
const char* ntpServer = "pool.ntp.org"; 
const long  gmtOffset_sec = 7 * 3600; // GMT+7 
const int   daylightOffset_sec = 0;     

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET); 

enum GameState {    
  MENU,    
  PLAYING_TETRIS,    
  TEXT_APP,    
  WIFI_SCAN,    
  WIFI_INPUT_PASS,    
  WIFI_CONNECTING,   
  BEACON_SPAM_APP,   
  CLOCK_APP,   
  TV_BGONE_BRANDS,   
  TV_BGONE_BUTTONS 
};

GameState currentState = MENU; 
int menuSelection = 0; // 0-5 

// ========================================================================= 
//                         TV-B-GONE VARIABLES 
// ========================================================================= 
enum TVBrand {   
  TV_SAMSUNG = 0,   
  TV_LG,   
  TV_SONY,   
  TV_TCL,   
  TV_PANASONIC 
};

enum TVButton {   
  TV_POWER = 0,   
  TV_UP,   
  TV_DOWN,   
  TV_LEFT,   
  TV_RIGHT,   
  TV_SELECT,   
  TV_BACK,   
  TV_HOME 
};

const int TV_BRAND_COUNT = 5; 
const int TV_BUTTON_COUNT = 8; 

const char* tvBrandNames[TV_BRAND_COUNT] = {   
  "SAMSUNG",   
  "LG",   
  "SONY",   
  "TCL",   
  "PANASONIC"
};

const char* tvButtonNames[TV_BUTTON_COUNT] = {   
  "POWER",   
  "UP",   
  "DOWN",   
  "LEFT",   
  "RIGHT",   
  "SELECT",   
  "BACK",   
  "HOME"
};

int tvBrandSelection = 0; 
int tvButtonSelection = 0; 
TVBrand selectedTVBrand = TV_SAMSUNG; 
bool tvSelectHolding = false; 
unsigned long tvSelectStart = 0; 
const unsigned long TV_HOLD_TIME = 5000; 

// Biến WiFi Scan 
int numNetworks = 0; 
int wifiSelection = 0; 
String selectedSSID = ""; 
String wifiPassword = ""; 
int wifiCharIndex = 0; 
const char charset[] = "abcdefghijklmnopqrstuvwxyz0123456789<"; 

// Chống rung nút
bool isPressed(uint8_t pin) {   
  if (digitalRead(pin) == LOW) {     
    delayMicroseconds(500);     
    return (digitalRead(pin) == LOW);   
  }
  return false; 
}

void waitForRelease(uint8_t pin) {   
  delay(20);   
  while (digitalRead(pin) == LOW) { delay(10); } 
}

// ========================================================================= 
//                   CẤU HÌNH & HÀM BEACON SPAMMER 
// ========================================================================= 
WebServer server(80); 
DNSServer dnsServer; 
Preferences preferences; 
const byte DNS_PORT = 53; 
const char* HOSTNAME = "KBeacon"; 

enum SpamMode {     
  MODE_VARIATION = 0,     
  MODE_LIST = 1,     
  MODE_FREEWIFI = 2 
};

String beaconMessage = "KBeacon"; 
String listMessages = ""; 
bool broadcasting = false;  
SpamMode currentMode = MODE_FREEWIFI; 
const int maxBeaconsPerChannel = 30; 
uint8_t beaconPacket[128]; 
uint8_t macAddr[6]; 
String current_ap_ssid = "KBeacon"; 
String current_ap_password = "KBpass123"; 

const char* symbols[] = {".", "_", " ", "  ", "   ", "..", "__", ". ", " .", "_ ", " _"}; 
const char* spaces[] = {"", " ", "  ", "   ", ".", "_"}; 
std::vector<String> customSSIDs; 

String freeWifiList[50]; 

void generate50FreeWiFiSSIDs() {   
  for (int i = 0; i < 50; i++) {     
    freeWifiList[i] = "freewifi " + String(i + 1); 
  }
}

const char index_html[] PROGMEM = R"rawliteral( 
<!DOCTYPE html> 
<html> 
<head>     
    <title>KBeacon Spammer</title>     
    <meta name="viewport" content="width=device-width, initial-scale=1">     
    <style>         
        body { font-family: Arial, sans-serif; padding: 20px; background-color: #f4f4f9; }         
        .button { background-color: #4CAF50; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; margin: 5px; text-decoration: none; display: inline-block; }         
        .stop { background-color: #f44336; }         
        .freewifi-btn { background-color: #ff9800; }         
        input[type=text], textarea { padding: 8px; margin: 8px 0; border: 1px solid #ccc; border-radius: 4px; width: 100%; max-width: 300px; }         
        .mode-container { border: 1px solid #ddd; padding: 15px; margin: 10px 0; border-radius: 4px; background: white; }     
    </style> 
</head> 
<body>     
    <h2>KBeacon Control Panel</h2>          
    <div class="mode-container">         
        <h3>Quick Action: 50x FreeWiFi Spam</h3>         
        <p>Phat 50 Beacon gia (SSID: <b>freewifi 1 -> 50</b>).</p>         
        <form action="/freewifi" method="POST">             
            <input type="submit" class="button freewifi-btn" value="Spam 50x FreeWiFi">         
        </form>     
    </div>     
    <div class="mode-container">         
        <h3>Mode 1: Variation Broadcasting</h3>         
        <form action="/update" method="POST">             
            <label>Base Message:</label><br>             
            <input type="text" name="message" value="%s"><br>             
            <input type="hidden" name="mode" value="variation">             
            <input type="submit" class="button" value="Update & Run Variation">         
        </form>     
    </div>     
    <div class="mode-container">         
        <h3>Mode 2: Custom List Broadcasting</h3>         
        <form action="/update" method="POST">             
            <label>Custom SSIDs (comma-separated):</label><br>             
            <textarea name="list" rows="3">%s</textarea><br>             
            <input type="hidden" name="mode" value="list">             
            <input type="submit" class="button" value="Update & Run List">         
        </form>     
    </div>     
    <div class="mode-container">         
        <h3>WiFi AP Settings</h3>         
        <form action="/updatewifi" method="POST">             
            <label>AP Name:</label><br>             
            <input type="text" name="ap_ssid" value="%s"><br>             
            <label>Password:</label><br>             
            <input type="text" name="ap_password" value="%s"><br>             
            <input type="submit" class="button" value="Update WiFi">         
        </form>     
    </div>     
    <br>     
    <a href="/toggle"><button class="button %s">%s Broadcasting</button></a>          
    <p>Status: <b>%s</b></p>     
    <p>Current Mode: <b>%s</b></p>     
    <p>Connected clients: %d</p> 
</body> 
</html> 
)rawliteral"; 

void blinkLED(int times, int delayTime = 100) {     
    for(int i = 0; i < times; i++) {         
        digitalWrite(LED_PIN, LOW);         
        delay(delayTime);         
        digitalWrite(LED_PIN, HIGH);         
        delay(delayTime);     
    }
}

void parseSSIDList(String list) {     
    customSSIDs.clear();     
    int start = 0;     
    int end = list.indexOf(',');     
    while (end >= 0) {         
        String ssid = list.substring(start, end);         
        ssid.trim();         
        if (ssid.length() > 0 && ssid.length() <= 32) {             
            customSSIDs.push_back(ssid);         
        }         
        start = end + 1;         
        end = list.indexOf(',', start);     
    }     
    String lastSSID = list.substring(start);     
    lastSSID.trim();     
    if (lastSSID.length() > 0 && lastSSID.length() <= 32) {         
        customSSIDs.push_back(lastSSID);     
    }
}

void generateRandomMac(uint8_t* mac) {     
    for(int i = 0; i < 6; i++) {         
        mac[i] = random(256);     
    }     
    mac[0] |= 0x02;     
    mac[0] &= 0xFE; 
}

void createBeaconPacket(const String& ssid) {     
    uint8_t packet[128] = {         
        0x80, 0x00, 0x00, 0x00,         
        0xff, 0xff, 0xff, 0xff, 0xff, 0xff,         
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00,         
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00,         
        0x00, 0x00,         
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,         
        0x64, 0x00,         
        0x11, 0x00,         
        0x00     
    };     
    memset(beaconPacket, 0, sizeof(beaconPacket));     
    memcpy(beaconPacket, packet, sizeof(packet));          
    generateRandomMac(macAddr);     
    memcpy(&beaconPacket[10], macAddr, 6);     
    memcpy(&beaconPacket[16], macAddr, 6);          
    beaconPacket[36] = 0x00;     
    beaconPacket[37] = ssid.length();     
    memcpy(&beaconPacket[38], ssid.c_str(), ssid.length()); 
}

String createVariation(int index) {     
    int spaceIndex = index % 6;     
    int symbolIndex = (index / 6) % 11;     
    return beaconMessage + spaces[spaceIndex] + symbols[symbolIndex]; 
}

void broadcastBeacon() {     
    if (!broadcasting) return;          
    static unsigned long lastLedToggle = 0;     
    const unsigned long LED_TOGGLE_INTERVAL = 100;     
    int channels[] = {1, 6, 11};     
    if (currentMode == MODE_FREEWIFI) {         
        for (int ch = 0; ch < 3; ch++) {             
            esp_wifi_set_channel(channels[ch], WIFI_SECOND_CHAN_NONE);             
            for (int i = 0; i < 50; i++) {                 
                createBeaconPacket(freeWifiList[i]);                 
                int packetSize = 38 + freeWifiList[i].length();                 
                for (int j = 0; j < maxBeaconsPerChannel; j++) {                     
                    esp_wifi_80211_tx(WIFI_IF_AP, beaconPacket, packetSize, false);                     
                    delayMicroseconds(10);                 
                }             
            }         
        }     
    }      
    else if (currentMode == MODE_VARIATION) {         
        for (int ch = 0; ch < 3; ch++) {             
            esp_wifi_set_channel(channels[ch], WIFI_SECOND_CHAN_NONE);             
            for (int i = 0; i < 50; i++) {                 
                String variation = createVariation(i);                 
                createBeaconPacket(variation);                 
                int packetSize = 38 + variation.length();                 
                for (int j = 0; j < maxBeaconsPerChannel; j++) {                     
                    esp_wifi_80211_tx(WIFI_IF_AP, beaconPacket, packetSize, false);                     
                    delayMicroseconds(10);                 
                }             
            }         
        }     
    }      
    else if (currentMode == MODE_LIST) {         
        for (int ch = 0; ch < 3; ch++) {             
            esp_wifi_set_channel(channels[ch], WIFI_SECOND_CHAN_NONE);             
            for (const String& ssid : customSSIDs) {                 
                createBeaconPacket(ssid);                 
                int packetSize = 38 + ssid.length();                 
                for (int j = 0; j < maxBeaconsPerChannel; j++) {                     
                    esp_wifi_80211_tx(WIFI_IF_AP, beaconPacket, packetSize, false);                     
                    delayMicroseconds(10);                 
                }             
            }         
        }     
    }          
    if (millis() - lastLedToggle >= LED_TOGGLE_INTERVAL) {         
        digitalWrite(LED_PIN, !digitalRead(LED_PIN));         
        lastLedToggle = millis();     
    } 
}

void handleRoot() {     
    char temp[3000];     
    String modeStr = "50x FreeWiFi Spam";     
    if (currentMode == MODE_VARIATION) modeStr = "Variation";     
    else if (currentMode == MODE_LIST) modeStr = "Custom List";     
    snprintf(temp, sizeof(temp), index_html,               
              beaconMessage.c_str(),              
              listMessages.c_str(),              
              current_ap_ssid.c_str(),              
              current_ap_password.c_str(),              
              broadcasting ? "stop" : "",              
              broadcasting ? "Stop" : "Start",              
              broadcasting ? "ACTIVE" : "STOPPED",              
              modeStr.c_str(),              
              WiFi.softAPgetStationNum());                   
    server.send(200, "text/html", temp); 
}

void handleFreeWiFi() {     
    currentMode = MODE_FREEWIFI;     
    broadcasting = true;     
    server.sendHeader("Location", "/");     
    server.send(303); 
}

void handleUpdate() {     
    if (server.hasArg("mode")) {         
        String mode = server.arg("mode");         
        if (mode == "variation" && server.hasArg("message")) {             
            beaconMessage = server.arg("message");             
            currentMode = MODE_VARIATION;             
            broadcasting = true;         
        } else if (mode == "list" && server.hasArg("list")) {             
            listMessages = server.arg("list");             
            parseSSIDList(listMessages);             
            currentMode = MODE_LIST;             
            broadcasting = true;         
        }     
    }     
    server.sendHeader("Location", "/");     
    server.send(303); 
}

void handleUpdateWifi() {     
    if (server.hasArg("ap_ssid") && server.hasArg("ap_password")) {         
        String new_ssid = server.arg("ap_ssid");         
        String new_password = server.arg("ap_password");                  
        if (new_ssid.length() < 1 || new_password.length() < 8) {             
            server.send(400, "text/plain", "SSID validation failed");             
            return;         
        }                  
        preferences.begin("wifi-config", false);         
        preferences.putString("ap_ssid", new_ssid);         
        preferences.putString("ap_password", new_password);         
        preferences.end();                  
        server.send(200, "text/plain", "WiFi updated. Restarting...");         
        delay(2000);         
        ESP.restart();     
    } 
}

void handleToggle() {     
    broadcasting = !broadcasting;     
    if (broadcasting) blinkLED(4);     
    else {         
        blinkLED(2);         
        digitalWrite(LED_PIN, HIGH);     
    }     
    server.sendHeader("Location", "/");     
    server.send(303); 
}

void handleNotFound() {     
    server.sendHeader("Location", String("http://") + WiFi.softAPIP().toString(), true);     
    server.send(302, "text/plain", ""); 
}

// ========================================================================= 
//                  HIỂN THỊ ICON SÓNG WIFI TRÊN OLED 
// ========================================================================= 
void drawWifiSignal(int x, int y) {   
  if (WiFi.status() == WL_CONNECTED) {     
    int32_t rssi = WiFi.RSSI();     
    display.fillRect(x, y + 6, 2, 2, SSD1306_WHITE);     
    if (rssi > -80) display.fillRect(x + 3, y + 4, 2, 4, SSD1306_WHITE);     
    if (rssi > -67) display.fillRect(x + 6, y + 2, 2, 6, SSD1306_WHITE);   
  } else {     
    display.drawLine(x, y + 2, x + 6, y + 8, SSD1306_WHITE);     
    display.drawLine(x + 6, y + 2, x, y + 8, SSD1306_WHITE); 
  }
}

// ========================================================================= 
//                         TV-B-GONE IR FUNCTIONS 
// ========================================================================= 
void blinkTVConfirm() {   
  digitalWrite(LED_PIN, LOW);   
  delay(120);   
  digitalWrite(LED_PIN, HIGH);   
  delay(120);   
  digitalWrite(LED_PIN, LOW);   
  delay(120);   
  digitalWrite(LED_PIN, HIGH); 
}

void sendSamsungTV(TVButton button) {   
  uint32_t code = 0;   
  switch (button) {     
    case TV_POWER:       code = 0xE0E040BF;       break;     
    case TV_UP:          code = 0xE0E006F9;       break;     
    case TV_DOWN:        code = 0xE0E08679;       break;     
    case TV_LEFT:        code = 0xE0E0A659;       break;     
    case TV_RIGHT:       code = 0xE0E046B9;       break;     
    case TV_SELECT:      code = 0xE0E016E9;       break;     
    case TV_BACK:        code = 0xE0E01AE5;       break;     
    case TV_HOME:        code = 0xE0E09E61;       break;   
  }
  if (code != 0) {     
    irsend.sendSAMSUNG(code, 32); 
  }
}

void sendLGTV(TVButton button) {   
  uint32_t code = 0;   
  switch (button) {     
    case TV_POWER:       code = 0x20DF10EF;       break;     
    case TV_UP:          code = 0x20DF02FD;       break;     
    case TV_DOWN:        code = 0x20DF827D;       break;     
    case TV_LEFT:        code = 0x20DFE01F;       break;     
    case TV_RIGHT:       code = 0x20DF609F;       break;     
    case TV_SELECT:      code = 0x20DF22DD;       break;     
    case TV_BACK:        code = 0x20DF14EB;       break;     
    case TV_HOME:        code = 0x20DF3EC1;       break;   
  }
  if (code != 0) {     
    irsend.sendLG(code, 32); 
  }
}

void sendSonyTV(TVButton button) {   
  uint16_t code = 0;   
  switch (button) {     
    case TV_POWER:       code = 0x0A90;       break;     
    case TV_UP:          code = 0x0074;       break;     
    case TV_DOWN:        code = 0x0075;       break;     
    case TV_LEFT:        code = 0x0025;       break;     
    case TV_RIGHT:       code = 0x0026;       break;     
    case TV_SELECT:      code = 0x0065;       break;     
    case TV_BACK:        code = 0x0062;       break;     
    case TV_HOME:        code = 0x0060;       break;   
  }
  if (code != 0) {     
    irsend.sendSony(code, 12, 2); 
  }
}

void sendTCLTV(TVButton button) {   
  uint32_t code = 0;   
  switch (button) {     
    case TV_POWER:       code = 0x20DF10EF;       break;     
    case TV_UP:          code = 0x20DF02FD;       break;     
    case TV_DOWN:        code = 0x20DF827D;       break;     
    case TV_LEFT:        code = 0x20DFE01F;       break;     
    case TV_RIGHT:       code = 0x20DF609F;       break;     
    case TV_SELECT:      code = 0x20DF22DD;       break;     
    case TV_BACK:        code = 0x20DF14EB;       break;     
    case TV_HOME:        code = 0x20DF3EC1;       break;   
  }
  if (code != 0) {     
    irsend.sendNEC(code, 32); 
  }
}

void sendPanasonicTV(TVButton button) {   
  uint32_t code = 0;   
  switch (button) {     
    case TV_POWER:       code = 0x40040100;       break;     
    case TV_UP:          code = 0x40040101;       break;     
    case TV_DOWN:        code = 0x40040102;       break;     
    case TV_LEFT:        code = 0x40040103;       break;     
    case TV_RIGHT:       code = 0x40040104;       break;     
    case TV_SELECT:      code = 0x40040105;       break;     
    case TV_BACK:        code = 0x40040106;       break;     
    case TV_HOME:        code = 0x40040107;       break;   
  }
  if (code != 0) {     
    irsend.sendPanasonic64(code); 
  }
}

void sendTVIR(TVBrand brand, TVButton button) {   
  switch (brand) {     
    case TV_SAMSUNG:   sendSamsungTV(button);   break;     
    case TV_LG:        sendLGTV(button);        break;     
    case TV_SONY:      sendSonyTV(button);      break;     
    case TV_TCL:       sendTCLTV(button);       break;     
    case TV_PANASONIC: sendPanasonicTV(button); break; 
  }
}

// ========================================================================= 
//                         TV-B-GONE BRAND MENU 
// ========================================================================= 
void updateTVBrandMenu() {   
  if (isPressed(BTN_UP)) {     
    tvBrandSelection--;     
    if (tvBrandSelection < 0)       
      tvBrandSelection = TV_BRAND_COUNT - 1;     
    waitForRelease(BTN_UP);   
  }
  if (isPressed(BTN_DOWN)) {     
    tvBrandSelection++;     
    if (tvBrandSelection >= TV_BRAND_COUNT)       
      tvBrandSelection = 0;     
    waitForRelease(BTN_DOWN);   
  }
  if (isPressed(BTN_LEFT)) {     
    currentState = MENU;     
    waitForRelease(BTN_LEFT);     
    return;   
  }
  if (isPressed(BTN_SELECT)) {     
    if (!tvSelectHolding) {       
      tvSelectHolding = true;       
      tvSelectStart = millis();     
    }     
    if (millis() - tvSelectStart >= TV_HOLD_TIME) {       
      selectedTVBrand = (TVBrand)tvBrandSelection;       
      tvButtonSelection = 0;       
      tvSelectHolding = false;       
      currentState = TV_BGONE_BUTTONS;       
      while (digitalRead(BTN_SELECT) == LOW) { delay(10); }       
      return;     
    }   
  } else {     
    tvSelectHolding = false;   
  }

  display.clearDisplay();   
  display.setTextSize(1);   
  display.setCursor(8, 0);   
  display.println(F("--- TV-B-GONE ---"));   
  int startIdx = 0;   
  if (tvBrandSelection >= 3)     
    startIdx = tvBrandSelection - 2;   
  for (int i = 0; i < 4; i++) {     
    int index = startIdx + i;     
    if (index >= TV_BRAND_COUNT) break;     
    int y = 14 + i * 11;     
    display.setCursor(16, y);     
    display.print(tvBrandNames[index]);     
    if (index == tvBrandSelection) {       
      display.setCursor(4, y);       
      display.print(F(">"));     
    }   
  }
  if (tvSelectHolding) {     
    unsigned long elapsed = millis() - tvSelectStart;     
    int remain = 5 - (elapsed / 1000);     
    if (remain < 0) remain = 0;     
    display.setCursor(0, 58);     
    display.print(F("GIU SELECT: "));     
    display.print(remain);     
    display.print(F("s"));   
  } else {     
    display.setCursor(0, 58);     
    display.print(F("SEL GIU 5S | LEFT"));   
  }
  display.display(); 
}

// ========================================================================= 
//                         TV-B-GONE BUTTON MENU 
// ========================================================================= 
void updateTVButtonMenu() {   
  if (isPressed(BTN_UP)) {     
    tvButtonSelection--;     
    if (tvButtonSelection < 0)       
      tvButtonSelection = TV_BUTTON_COUNT - 1;     
    waitForRelease(BTN_UP);   
  }
  if (isPressed(BTN_DOWN)) {     
    tvButtonSelection++;     
    if (tvButtonSelection >= TV_BUTTON_COUNT)       
      tvButtonSelection = 0;     
    waitForRelease(BTN_DOWN);   
  }
  if (isPressed(BTN_LEFT)) {     
    currentState = TV_BGONE_BRANDS;     
    tvSelectHolding = false;     
    waitForRelease(BTN_LEFT);     
    return;   
  }
  if (isPressed(BTN_SELECT)) {     
    if (!tvSelectHolding) {       
      tvSelectHolding = true;       
      tvSelectStart = millis();     
    }     
    unsigned long elapsed = millis() - tvSelectStart;     
    if (elapsed >= TV_HOLD_TIME) {       
      while (digitalRead(BTN_SELECT) == LOW) {         
        display.clearDisplay();         
        display.setTextSize(1);         
        display.setCursor(20, 20);         
        display.println(F("SAN SANG PHAT IR"));         
        display.setCursor(30, 40);         
        display.println(F("THA SELECT"));         
        display.display();         
        delay(10);       
      }       
      tvSelectHolding = false;       
      blinkTVConfirm();       
      sendTVIR(selectedTVBrand, (TVButton)tvButtonSelection);       
      currentState = TV_BGONE_BUTTONS;       
      return;     
    }   
  } else {     
    tvSelectHolding = false;   
  }

  display.clearDisplay();   
  display.setTextSize(1);   
  display.setCursor(5, 0);   
  display.print(F("--- "));   
  display.print(tvBrandNames[selectedTVBrand]);   
  display.println(F(" ---"));   
  int startIdx = 0;   
  if (tvButtonSelection >= 4)     
    startIdx = tvButtonSelection - 3;   
  for (int i = 0; i < 4; i++) {     
    int index = startIdx + i;     
    if (index >= TV_BUTTON_COUNT) break;     
    int y = 13 + i * 11;     
    display.setCursor(17, y);     
    display.print(tvButtonNames[index]);     
    if (index == tvButtonSelection) {       
      display.setCursor(4, y);       
      display.print(F(">"));     
    }   
  }
  if (tvSelectHolding) {     
    unsigned long elapsed = millis() - tvSelectStart;     
    int seconds = elapsed / 1000;     
    if (seconds > 5) seconds = 5;     
    display.setCursor(0, 58);     
    display.print(F("GIU SELECT "));     
    display.print(seconds);     
    display.print(F("/5s"));   
  } else {     
    display.setCursor(0, 58);     
    display.print(F("SEL 5S = IR"));   
  }
  display.display(); 
}

// ========================================================================= 
//                                1. GAME TETRIS 
// ========================================================================= 
const int TETRIS_W = 10, TETRIS_H = 20, BLOCK_SIZE = 3; 
const int TETRIS_X = 2, TETRIS_Y = 2; 
byte tetrisBoard[TETRIS_H][TETRIS_W]; 
const byte shapes[4][4][4] = {   
  {{0,0,0,0},{1,1,1,1},{0,0,0,0},{0,0,0,0}},   
  {{0,1,1,0},{0,1,1,0},{0,0,0,0},{0,0,0,0}},   
  {{0,1,0,0},{1,1,1,0},{0,0,0,0},{0,0,0,0}},   
  {{1,0,0,0},{1,1,1,0},{0,0,0,0},{0,0,0,0}}
};

int tX = 3, tY = 0, tType = 0; 
byte tShape[4][4]; 
unsigned long tLastDrop = 0; 
int tScore = 0; 
bool tGameOver = false; 

bool checkCollision(int posX, int posY, byte shape[4][4]) {   
  for (int r = 0; r < 4; r++) {     
    for (int c = 0; c < 4; c++) {       
      if (shape[r][c]) {         
        int bX = posX + c, bY = posY + r;         
        if (bX < 0 || bX >= TETRIS_W || bY >= TETRIS_H) return true;         
        if (bY >= 0 && tetrisBoard[bY][bX]) return true;       
      }     
    }   
  }
  return false; 
}

void rotateTetrisPiece() {   
  byte temp[4][4];   
  for (int r = 0; r < 4; r++)     
    for (int c = 0; c < 4; c++) temp[r][c] = tShape[3 - c][r];   
  if (!checkCollision(tX, tY, temp)) memcpy(tShape, temp, sizeof(temp)); 
}

void spawnTetrisPiece() {   
  tX = 3; tY = 0; tType = esp_random() % 4;   
  for (int r = 0; r < 4; r++) for (int c = 0; c < 4; c++) tShape[r][c] = shapes[tType][r][c];   
  if (checkCollision(tX, tY, tShape)) tGameOver = true; 
}

void initTetris() {   
  memset(tetrisBoard, 0, sizeof(tetrisBoard));   
  tScore = 0; tGameOver = false; spawnTetrisPiece(); 
}

void updateTetris() {   
  if (tGameOver) {     
    display.clearDisplay();     
    display.setTextSize(2); display.setCursor(10, 15); display.println(F("GAME OVER"));     
    display.setTextSize(1); display.setCursor(25, 40); display.print(F("Diem: ")); display.print(tScore);     
    display.display(); return;   
  }
  if (isPressed(BTN_LEFT))  { if (!checkCollision(tX - 1, tY, tShape)) tX--; delay(100); }   
  if (isPressed(BTN_RIGHT)) { if (!checkCollision(tX + 1, tY, tShape)) tX++; delay(100); }   
  if (isPressed(BTN_SELECT) || isPressed(BTN_UP)) { rotateTetrisPiece(); waitForRelease(BTN_SELECT); }   
  if (isPressed(BTN_DOWN)) {     
    while (!checkCollision(tX, tY + 1, tShape)) tY++;     
    for (int r = 0; r < 4; r++) for (int c = 0; c < 4; c++) if (tShape[r][c] && (tY + r >= 0)) tetrisBoard[tY + r][tX + c] = 1;     
    for (int r = TETRIS_H - 1; r >= 0; r--) {       
      bool full = true;       
      for (int c = 0; c < TETRIS_W; c++) if (!tetrisBoard[r][c]) { full = false; break; }       
      if (full) {         
        tScore += 10;         
        for (int mR = r; mR > 0; mR--) memcpy(tetrisBoard[mR], tetrisBoard[mR - 1], TETRIS_W);         
        memset(tetrisBoard[0], 0, TETRIS_W); r++;       
      }     
    }     
    spawnTetrisPiece(); delay(150);   
  }
  if (millis() - tLastDrop > 400) {     
    tLastDrop = millis();     
    if (!checkCollision(tX, tY + 1, tShape)) tY++;     
    else {       
      for (int r = 0; r < 4; r++) for (int c = 0; c < 4; c++) if (tShape[r][c] && (tY + r >= 0)) tetrisBoard[r + tY][c + tX] = 1;       
      spawnTetrisPiece();     
    }   
  }
  display.clearDisplay();   
  drawWifiSignal(118, 0);   
  display.drawRect(TETRIS_X - 1, TETRIS_Y - 1, (TETRIS_W * BLOCK_SIZE) + 2, (TETRIS_H * BLOCK_SIZE) + 2, SSD1306_WHITE);   
  for (int r = 0; r < TETRIS_H; r++)     
    for (int c = 0; c < TETRIS_W; c++)       
      if (tetrisBoard[r][c]) display.fillRect(TETRIS_X + c * BLOCK_SIZE, TETRIS_Y + r * BLOCK_SIZE, BLOCK_SIZE - 1, BLOCK_SIZE - 1, SSD1306_WHITE);   
  for (int r = 0; r < 4; r++)     
    for (int c = 0; c < 4; c++)       
      if (tShape[r][c] && (tY + r >= 0)) display.fillRect(TETRIS_X + (tX + c) * BLOCK_SIZE, TETRIS_Y + (tY + r) * BLOCK_SIZE, BLOCK_SIZE - 1, BLOCK_SIZE - 1, SSD1306_WHITE);   
  display.setTextSize(1); display.setCursor(45, 5); display.print(F("TETRIS"));   
  display.setCursor(45, 20); display.print(F("Score:")); display.setCursor(45, 32); display.print(tScore);   
  display.display(); 
}

// ========================================================================= 
//                                2. TEXT APP 
// ========================================================================= 
String userText = ""; 
int charIndex = 0; 

void initTextApp() { userText = ""; charIndex = 0; } 

void updateTextApp() {   
  int charsetSize = sizeof(charset) - 1;   
  if (isPressed(BTN_UP)) { charIndex = (charIndex + 1) % charsetSize; waitForRelease(BTN_UP); }   
  if (isPressed(BTN_DOWN)) { charIndex = (charIndex - 1 + charsetSize) % charsetSize; waitForRelease(BTN_DOWN); }   
  if (isPressed(BTN_SELECT)) {     
    if (charset[charIndex] == '<') {        
      if (userText.length() > 0) userText.remove(userText.length() - 1);      
    } else if (userText.length() < 61) {       
      userText += charset[charIndex];     
    }     
    waitForRelease(BTN_SELECT);   
  }
  if (isPressed(BTN_RIGHT)) {     
    if (userText.length() < 61) userText += ' ';     
    waitForRelease(BTN_RIGHT);   
  }
  if (isPressed(BTN_LEFT)) {      
    if (userText.length() > 0) userText.remove(userText.length() - 1);      
    else { currentState = MENU; }     
    waitForRelease(BTN_LEFT);    
  }
  display.clearDisplay();   
  drawWifiSignal(118, 0);   
  display.setTextSize(1); display.setCursor(0, 0);   
  display.println(userText);   
  display.drawFastHLine(0, 42, 128, SSD1306_WHITE);   
  display.setCursor(0, 48); display.print(F("Ky tu: [ "));   
  display.setTextSize(2); display.setCursor(60, 46);   
  display.print(charset[charIndex]);   
  display.setTextSize(1); display.setCursor(85, 48); display.print(F(" ]"));   
  display.display(); 
}

// ========================================================================= 
//                               3. QUÉT WIFI 
// ========================================================================= 
void startWifiScan() {   
  display.clearDisplay();   
  drawWifiSignal(118, 0);   
  display.setTextSize(1);   
  display.setCursor(0, 20);   
  display.println(F("Dang quet WiFi..."));   
  display.display();   
  WiFi.mode(WIFI_AP_STA);   
  numNetworks = WiFi.scanNetworks();   
  wifiSelection = 0;   
  currentState = WIFI_SCAN; 
}

void updateWifiScan() {   
  if (numNetworks == 0) {     
    display.clearDisplay();     
    drawWifiSignal(118, 0);     
    display.setCursor(0, 20);     
    display.println(F("Khong tim thay WiFi!"));     
    display.setCursor(0, 40);     
    display.println(F("Nhan UP de quet lai"));     
    display.display();     
    if (isPressed(BTN_UP)) { startWifiScan(); waitForRelease(BTN_UP); }     
    if (isPressed(BTN_LEFT)) { currentState = MENU; waitForRelease(BTN_LEFT); }     
    return;   
  }
  if (isPressed(BTN_UP)) {     
    wifiSelection = (wifiSelection - 1 + numNetworks) % numNetworks;     
    waitForRelease(BTN_UP);   
  }
  if (isPressed(BTN_DOWN)) {     
    wifiSelection = (wifiSelection + 1) % numNetworks;     
    waitForRelease(BTN_DOWN);   
  }
  if (isPressed(BTN_LEFT)) {     
    currentState = MENU;     
    waitForRelease(BTN_LEFT);     
    return;   
  }
  if (isPressed(BTN_SELECT)) {     
    selectedSSID = WiFi.SSID(wifiSelection);     
    wifiPassword = "";     
    wifiCharIndex = 0;     
    currentState = WIFI_INPUT_PASS;     
    waitForRelease(BTN_SELECT);     
    return;   
  }
  display.clearDisplay();   
  drawWifiSignal(118, 0);   
  display.setTextSize(1);   
  display.setCursor(0, 0);   
  display.print(F("WiFi (")); display.print(numNetworks); display.print(F(")"));   
  int startIdx = (wifiSelection > 3) ? wifiSelection - 3 : 0;   
  for (int i = 0; i < 4; i++) {     
    int itemIdx = startIdx + i;     
    if (itemIdx < numNetworks) {       
      int yPos = 14 + (i * 12);       
      display.setCursor(10, yPos);       
      String ssidStr = WiFi.SSID(itemIdx);       
      if (ssidStr.length() > 14) ssidStr = ssidStr.substring(0, 14);       
      display.print(ssidStr);       
      if (itemIdx == wifiSelection) {         
        display.setCursor(0, yPos);         
        display.print(F(">"));       
      }     
    }   
  }
  display.display(); 
}

void updateWifiInputPass() {   
  int charsetSize = sizeof(charset) - 1;   
  if (isPressed(BTN_UP)) { wifiCharIndex = (wifiCharIndex + 1) % charsetSize; waitForRelease(BTN_UP); }   
  if (isPressed(BTN_DOWN)) { wifiCharIndex = (wifiCharIndex - 1 + charsetSize) % charsetSize; waitForRelease(BTN_DOWN); }   
  if (isPressed(BTN_RIGHT)) {     
    if (wifiPassword.length() < 32) wifiPassword += ' ';     
    waitForRelease(BTN_RIGHT);   
  }
  if (isPressed(BTN_LEFT)) {     
    if (wifiPassword.length() > 0) {       
      wifiPassword.remove(wifiPassword.length() - 1);     
    } else {       
      currentState = WIFI_SCAN;     
    }     
    waitForRelease(BTN_LEFT);   
  }
  if (digitalRead(BTN_SELECT) == LOW) {     
    unsigned long pressTime = millis();     
    bool longPressed = false;     
    while (digitalRead(BTN_SELECT) == LOW) {       
      if (millis() - pressTime >= 5000) {         
        longPressed = true;         
        digitalWrite(LED_PIN, (millis() / 100) % 2 ? LOW : HIGH);        
      }       
      delay(10);     
    }     
    digitalWrite(LED_PIN, HIGH);     
    if (longPressed) {       
      currentState = WIFI_CONNECTING;       
      return;     
    } else {       
      if (charset[wifiCharIndex] == '<') {         
        if (wifiPassword.length() > 0) wifiPassword.remove(wifiPassword.length() - 1);       
      } else if (wifiPassword.length() < 32) {         
        wifiPassword += charset[wifiCharIndex];       
      }     
    }   
  }
  display.clearDisplay();   
  drawWifiSignal(118, 0);   
  display.setTextSize(1);   
  display.setCursor(0, 0);   
  display.print(F("SSID: ")); display.println(selectedSSID.substring(0, 10));   
  display.setCursor(0, 12);   
  display.print(F("MK: "));   
  display.println(wifiPassword);   
  display.drawFastHLine(0, 36, 128, SSD1306_WHITE);   
  display.setCursor(0, 42); display.print(F("Chon: [ "));   
  display.setTextSize(2); display.setCursor(50, 40);   
  display.print(charset[wifiCharIndex]);   
  display.setTextSize(1); display.setCursor(70, 42); display.print(F(" ]"));   
  display.setCursor(0, 56);   
  display.print(F("Giu SELECT 5s: Ket noi"));   
  display.display(); 
}

void connectToWifi() {   
  display.clearDisplay();   
  drawWifiSignal(118, 0);   
  display.setTextSize(1);   
  display.setCursor(0, 10);   
  display.println(F("Dang ket noi..."));   
  display.setCursor(0, 25);   
  display.println(selectedSSID);   
  display.display();   
  WiFi.begin(selectedSSID.c_str(), wifiPassword.c_str());   
  int timeout = 0;   
  while (WiFi.status() != WL_CONNECTED && timeout < 20) {     
    delay(500);     
    timeout++;   
  }
  display.clearDisplay();   
  drawWifiSignal(118, 0);   
  display.setCursor(0, 20);   
  if (WiFi.status() == WL_CONNECTED) {     
    display.println(F("KET NOI THANH CONG!"));     
    display.setCursor(0, 35);     
    display.print(F("IP: ")); display.println(WiFi.localIP());          
    configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);   
  } else {     
    display.println(F("KET NOI THAT BAI!"));   
  }
  display.display();   
  delay(2000);   
  currentState = MENU; 
}

// ========================================================================= 
//                                4. BEACON SPAM APP 
// ========================================================================= 
void startBeaconSpamApp() {   
  currentMode = MODE_FREEWIFI;   
  broadcasting = true;   
  blinkLED(3, 80);   
  currentState = BEACON_SPAM_APP; 
}

void updateBeaconSpamApp() {   
  if (isPressed(BTN_LEFT)) {     
    broadcasting = false;      
    digitalWrite(LED_PIN, HIGH);     
    currentState = MENU;     
    waitForRelease(BTN_LEFT);     
    return;   
  }
  display.clearDisplay();   
  display.setTextSize(1);   
  display.setCursor(0, 0);   
  display.println(F("--- BEACON SPAMMER ---"));   
  display.setCursor(0, 14);   
  display.print(F("Mode: ")); display.println(F("50x freewifi"));   
  display.setCursor(0, 26);   
  display.print(F("AP Control: ")); display.println(current_ap_ssid);   
  display.setCursor(0, 38);   
  display.print(F("IP WebUI: ")); display.println(WiFi.softAPIP().toString());   
  display.setCursor(0, 52);   
  display.println(F("Nhan LEFT de THOAT"));   
  display.display(); 
}

// ========================================================================= 
//                  5. CLOCK APP 
// ========================================================================= 
void updateClockApp() {   
  if (isPressed(BTN_LEFT)) {     
    currentState = MENU;     
    waitForRelease(BTN_LEFT);     
    return;   
  }
  display.clearDisplay();   
  drawWifiSignal(118, 0);   
  if (WiFi.status() != WL_CONNECTED) {     
    display.setTextSize(1);     
    display.setCursor(0, 15);     
    display.println(F("Chua ket noi WiFi!"));     
    display.setCursor(0, 35);     
    display.println(F("Vui long ket noi WiFi"));     
    display.setCursor(0, 47);     
    display.println(F("de dong bo gio."));   
  } else {     
    struct tm timeinfo;     
    if (!getLocalTime(&timeinfo, 3000)) {        
      configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);       
      display.setTextSize(1);       
      display.setCursor(0, 25);       
      display.println(F("Dang lay gio NTP..."));     
    } else {       
      char dateBuff[16];       
      snprintf(dateBuff, sizeof(dateBuff), "%02d/%02d/%04d",                 
                timeinfo.tm_mday, timeinfo.tm_mon + 1, timeinfo.tm_year + 1900);       
      char timeBuff[10];       
      snprintf(timeBuff, sizeof(timeBuff), "%02d:%02d",                 
                timeinfo.tm_hour, timeinfo.tm_min);       
      display.setTextSize(1);       
      display.setCursor(30, 12);       
      display.println(dateBuff);       
      display.setTextSize(2);       
      display.setCursor(34, 30);       
      display.println(timeBuff);       
      display.setTextSize(1);       
      display.setCursor(42, 52);       
      display.println(F("(GMT+7)"));     
    }   
  }
  display.display(); 
}

// ========================================================================= 
//                            SETUP & MAIN LOOP 
// ========================================================================= 
void setup() {   
  randomSeed(esp_random());   
  Serial.begin(115200);   
  Wire.begin(I2C_SDA, I2C_SCL);   
  pinMode(BTN_LEFT, INPUT_PULLUP);   
  pinMode(BTN_RIGHT, INPUT_PULLUP);   
  pinMode(BTN_SELECT, INPUT_PULLUP);   
  pinMode(BTN_DOWN, INPUT_PULLUP);   
  pinMode(BTN_UP, INPUT_PULLUP);   
  pinMode(LED_PIN, OUTPUT);   
  digitalWrite(LED_PIN, HIGH); 
  pinMode(IR_LED_PIN, OUTPUT); 
  digitalWrite(IR_LED_PIN, LOW); 
  irsend.begin();   

  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) for (;;);   
  display.clearDisplay();   
  display.setTextColor(SSD1306_WHITE);   
  generate50FreeWiFiSSIDs();   
  preferences.begin("wifi-config", false);   
  current_ap_ssid = preferences.getString("ap_ssid", "KBeacon");   
  current_ap_password = preferences.getString("ap_password", "KBpass123");   
  preferences.end();   
  WiFi.disconnect();   
  WiFi.mode(WIFI_AP_STA);   
  esp_wifi_set_mode(WIFI_MODE_APSTA);   
  WiFi.softAP(current_ap_ssid.c_str(), current_ap_password.c_str(), 6);   
  MDNS.begin(HOSTNAME);   
  dnsServer.start(DNS_PORT, "*", WiFi.softAPIP());   
  server.onNotFound(handleNotFound);   
  server.on("/", handleRoot);   
  server.on("/freewifi", handleFreeWiFi);   
  server.on("/update", handleUpdate);   
  server.on("/updatewifi", handleUpdateWifi);   
  server.on("/toggle", handleToggle);   
  server.begin();   
  broadcasting = false;   
  blinkLED(2, 100); 
}

void loop() {   
  dnsServer.processNextRequest();   
  server.handleClient();   
  broadcastBeacon();   

  if (currentState == MENU) {     
    if (isPressed(BTN_UP)) {       
      menuSelection = (menuSelection - 1 + 6) % 6;       
      waitForRelease(BTN_UP);     
    }     
    if (isPressed(BTN_DOWN)) {       
      menuSelection = (menuSelection + 1) % 6;       
      waitForRelease(BTN_DOWN);     
    }     
    if (isPressed(BTN_SELECT)) {       
      if (menuSelection == 0)      { initTetris(); currentState = PLAYING_TETRIS; }       
      else if (menuSelection == 1) { initTextApp(); currentState = TEXT_APP; }       
      else if (menuSelection == 2) { startWifiScan(); }       
      else if (menuSelection == 3) { startBeaconSpamApp(); }        
      else if (menuSelection == 4) { currentState = CLOCK_APP; }       
      else if (menuSelection == 5) {         
        tvBrandSelection = 0;         
        tvButtonSelection = 0;         
        tvSelectHolding = false;         
        currentState = TV_BGONE_BRANDS;       
      }
      waitForRelease(BTN_SELECT);       
      return;     
    }     
    display.clearDisplay();     
    drawWifiSignal(118, 0);          
    display.setTextSize(1);     
    display.setCursor(10, 0); display.println(F("--- MAIN MENU ---"));     
    const char* menuNames[] = {   
      "1. TETRIS",   
      "2. TEXT APP",   
      "3. QUET WIFI",   
      "4. BEACON SPAM",   
      "5. DONG HO",   
      "6. TV-B-GONE"     
    };
    int startIdx = (menuSelection > 3) ? menuSelection - 3 : 0;     
    for (int i = 0; i < 4; i++) {       
      int itemIdx = startIdx + i;       
      if (itemIdx < 6) {         
        int yPos = 14 + (i * 12);         
        display.setCursor(15, yPos);         
        display.print(menuNames[itemIdx]);         
        if (itemIdx == menuSelection) {           
          display.setCursor(3, yPos);           
          display.print(F(">"));         
        }       
      }     
    }     
    display.display();   
  }    
  else if (currentState == PLAYING_TETRIS)   { updateTetris(); }    
  else if (currentState == TEXT_APP)         { updateTextApp(); }   
  else if (currentState == WIFI_SCAN)        { updateWifiScan(); }   
  else if (currentState == WIFI_INPUT_PASS)  { updateWifiInputPass(); }   
  else if (currentState == WIFI_CONNECTING)  { connectToWifi(); }   
  else if (currentState == BEACON_SPAM_APP)  { updateBeaconSpamApp(); } 
  else if (currentState == CLOCK_APP)        { updateClockApp(); } 
  else if (currentState == TV_BGONE_BRANDS)  { updateTVBrandMenu(); } 
  else if (currentState == TV_BGONE_BUTTONS) { updateTVButtonMenu(); } 
}
