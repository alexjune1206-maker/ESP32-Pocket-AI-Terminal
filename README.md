#include <WiFi.h>
#include <HTTPClient.h>
#include <WiFiClientSecure.h>
#include <Adafruit_GFX.h>
#include <Adafruit_ST7735.h>
#include <SPI.h>
#include <TJpg_Decoder.h>
#include <ArduinoJson.h>

#define TFT_CS    5
#define TFT_RST   4
#define TFT_DC    2
#define TFT_MOSI  23
#define TFT_SCLK  18

#define JOY_X     34
#define JOY_Y     35
#define JOY_SEL   32

const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";
const char* groq_api_key = "YOUR_GROQ_API_KEY";

Adafruit_ST7735 tft = Adafruit_ST7735(TFT_CS, TFT_DC, TFT_MOSI, TFT_SCLK, TFT_RST);

char alphabet[] = " ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
int alphaLength = sizeof(alphabet) - 1;
int currentLetterIdx = 0;
String currentPrompt = "";

unsigned long lastMoveTime = 0;
const int moveDelay = 200;

uint8_t* imageBuffer = NULL;
size_t bufferSize = 0;

bool tft_output(int16_t x, int16_t y, uint16_t w, uint16_t h, uint16_t* bitmap) {
  if (y >= tft.height()) return false;
  tft.drawRGBBitmap(x, y, bitmap, w, h);
  return true;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(JOY_SEL, INPUT_PULLUP);
  
  tft.initR_BlackTab();
  tft.setRotation(1);
  tft.fillScreen(ST7735_BLACK);
  tft.setTextColor(ST7735_GREEN, ST7735_BLACK);
  tft.setTextSize(1);
  
  tft.println("Connecting to WiFi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    tft.print(".");
  }
  
  tft.fillScreen(ST7735_BLACK);
  tft.setCursor(0, 0);
  tft.println("WiFi Connected!");
  delay(1000);
  
  TJpg_Decoder.setJpgScale(1);
  TJpg_Decoder.setCallback(tft_output);
  
  updateDisplay();
}

void loop() {
  int xVal = analogRead(JOY_X);
  int yVal = analogRead(JOY_Y);
  int btnSel = digitalRead(JOY_SEL);
  
  if (millis() - lastMoveTime > moveDelay) {
    if (xVal < 1500) {
      currentLetterIdx = (currentLetterIdx + 1) % alphaLength;
      updateDisplay();
      lastMoveTime = millis();
    } else if (xVal > 2500) {
      currentLetterIdx = (currentLetterIdx - 1 + alphaLength) % alphaLength;
      updateDisplay();
      lastMoveTime = millis();
    }
    
    if (yVal < 1500) {
      if (alphabet[currentLetterIdx] != ' ') {
        currentPrompt += alphabet[currentLetterIdx];
      } else if (currentPrompt.length() > 0 && currentPrompt[currentPrompt.length()-1] != ' ') {
        currentPrompt += " ";
      }
      currentLetterIdx = 0;
      updateDisplay();
      lastMoveTime = millis();
    } else if (yVal > 2500) {
      if (currentPrompt.length() > 0) {
        currentPrompt.remove(currentPrompt.length() - 1);
        updateDisplay();
      }
      lastMoveTime = millis();
    }
  }
  
  if (btnSel == LOW) {
    delay(50);
    if (digitalRead(JOY_SEL) == LOW) {
      currentPrompt.trim();
      if (currentPrompt.length() > 0) {
        processAIRequest(currentPrompt);
        currentPrompt = "";
        currentLetterIdx = 0;
        updateDisplay();
      }
      while(digitalRead(JOY_SEL) == LOW);
    }
  }
  delay(10);
}

void updateDisplay() {
  tft.fillScreen(ST7735_BLACK);
  tft.setCursor(0, 0);
  tft.setTextColor(ST7735_GREEN);
  tft.println("=== AI TERMINAL ===");
  tft.println();
  
  tft.setTextColor(ST7735_WHITE);
  tft.print("Prompt: ");
  tft.println(currentPrompt);
  tft.println();
  
  tft.setTextColor(ST7735_YELLOW);
  tft.print("Char: [ ");
  if (alphabet[currentLetterIdx] == ' ') {
    tft.print("_");
  } else {
    tft.print(alphabet[currentLetterIdx]);
  }
  tft.println(" ]");
  
  tft.setTextColor(ST7735_BLUE);
  tft.println();
  tft.println("X: L/R Char | Y Up: Add");
  tft.println("Y Down: Del | BTN: Send");
}

void processAIRequest(String prompt) {
  tft.fillScreen(ST7735_BLACK);
  tft.setCursor(0, 0);
  tft.setTextColor(ST7735_MAGENTA);
  tft.println("Optimizing prompt...");
  
  String EnglishPrompt = translateAndOptimize(prompt);
  if(EnglishPrompt == "") {
    EnglishPrompt = prompt;
  }
  
  tft.fillScreen(ST7735_BLACK);
  tft.setCursor(0, 0);
  tft.println("Generating image...");
  
  String encodedPrompt = urlEncode(EnglishPrompt);
  String imageUrl = "https://image.pollinations.ai/p/" + encodedPrompt + "?width=128&height=128&model=turbo&seed=" + String(random(1, 99999));
  
  downloadAndDisplayImage(imageUrl);
}

String translateAndOptimize(String text) {
  WiFiClientSecure client;
  client.setInsecure();
  HTTPClient http;
  
  if (http.begin(client, "https://api.groq.com/openai/v1/chat/completions")) {
    http.addHeader("Content-Type", "application/json");
    String authHeader = "Bearer " + String(groq_api_key);
    http.addHeader("Authorization", authHeader.c_str());
    
    StaticJsonDocument<512> doc;
    doc["model"] = "llama3-8b-8192";
    JsonArray messages = doc.createNestedArray("messages");
    
    JsonObject msg1 = messages.createNestedObject();
    msg1["role"] = "system";
    msg1["content"] = "Translate the user prompt to English if it is in Romanian. Expand it into a short, descriptive image prompt for a digital art style. Output ONLY the English prompt text. No quotes, no intro, no explanation.";
    
    JsonObject msg2 = messages.createNestedObject();
    msg2["role"] = "user";
    msg2["content"] = text;
    
    String requestBody;
    serializeJson(doc, requestBody);
    
    int httpResponseCode = http.POST(requestBody);
    if (httpResponseCode == HTTP_CODE_OK) {
      String response = http.getString();
      StaticJsonDocument<1024> respDoc;
      deserializeJson(respDoc, response);
      String result = respDoc["choices"][0]["message"]["content"].as<String>();
      result.trim();
      http.end();
      return result;
    }
    http.end();
  }
  return "";
}

void downloadAndDisplayImage(String url) {
  WiFiClientSecure client;
  client.setInsecure();
  HTTPClient http;
  
  if (http.begin(client, url)) {
    int httpResponseCode = http.GET();
    if (httpResponseCode == HTTP_CODE_OK) {
      int contentLength = http.getSize();
      if (contentLength > 0) {
        if(imageBuffer != NULL) {
          free(imageBuffer);
          imageBuffer = NULL;
        }
        
        imageBuffer = (uint8_t*)malloc(contentLength);
        if (imageBuffer != NULL) {
          WiFiClient* stream = http.getStreamPtr();
          size_t bytesRead = 0;
          while (http.connected() && bytesRead < contentLength) {
            size_t available = stream->available();
            if (available) {
              int readNow = stream->readBytes(&imageBuffer[bytesRead], available);
              bytesRead += readNow;
            }
            delay(1);
          }
          
          tft.fillScreen(ST7735_BLACK);
          TJpg_Decoder.drawJpg(0, 0, imageBuffer, bytesRead);
          delay(25000);
        }
      }
    } else {
      tft.fillScreen(ST7735_BLACK);
      tft.setCursor(0, 0);
      tft.setTextColor(ST7735_RED);
      tft.print("HTTP Error: ");
      tft.println(httpResponseCode);
      delay(3000);
    }
    http.end();
  }
  
  if(imageBuffer != NULL) {
    free(imageBuffer);
    imageBuffer = NULL;
  }
}

String urlEncode(String str) {
  String encodedString = "";
  char c;
  char charBuf[5];
  for (int i = 0; i < str.length(); i++) {
    c = str.charAt(i);
    if (isalnum(c)) {
      encodedString += c;
    } else if (c == ' ') {
      encodedString += "%20";
    } else {
      sprintf(charBuf, "%%%02X", c);
      encodedString += charBuf;
    }
  }
  return encodedString;
}
