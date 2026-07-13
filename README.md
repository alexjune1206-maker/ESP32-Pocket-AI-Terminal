# ESP32-Pocket-AI-Terminal
Standalone ESP32 AI image generator using a joystick and ST7735 display.
# ESP32 Pocket AI Terminal 📟✨

A compact, cyberpunk-style standalone terminal built on the ESP32 micro-controller. It allows users to compose text prompts using a physical analog joystick and hardware keyboard interface to generate AI digital art in real-time, completely bypassing the need for a computer.

Developed by **voidpos** 🪐

---

## 🚀 Features
- **Standalone Input:** Full alphanumeric layout selectable using a standard 2-axis analog joystick.
- **Dynamic AI Generation:** Connects directly via secure Wi-Fi (`HTTPS`) to LLM & Diffusion servers.
- **Smart Optimization:** Translates and expands user prompts dynamically into highly descriptive English scenes using the Llama 3 model.
- **Ultra-Fast Rendering:** Leverages the Pollinations AI Turbo model to return and display high-contrast 128x128 color graphics onto an ST7735 SPI display within seconds using hard-allocated JPEG stream decoding.

---

## 🛠️ Hardware Requirements
| Component | Description |
| :--- | :--- |
| **ESP32** | NodeMCU / ESP32-WROOM-32 Development Board |
| **ST7735 Display** | 1.44" or 1.8" SPI TFT Color Screen (128x128 or 128x160) |
| **Joystick Module** | Standard Dual-Axis Analog Joystick with integrated Push-Button |

---

## 🔌 Pin Configuration
Ensure your hardware layout matches the following SPI and Analog assignments:

### TFT Display (SPI)
- `CS`   -> GPIO 5
- `RST`  -> GPIO 4
- `DC`   -> GPIO 2
- `SDA`  -> GPIO 23 (MOSI)
- `SCL`  -> GPIO 18 (SCLK)

### Joystick
- `X-Axis` -> GPIO 34
- `Y-Axis` -> GPIO 35
- `Switch` -> GPIO 32 (SEL / Push-Button)

---

## 💻 Setup & Installation

1. Clone or download this repository.
2. Open the main `.ino` file in the Arduino IDE.
3. Make sure you have installed the required libraries: `WiFi`, `HTTPClient`, `Adafruit_GFX`, `Adafruit_ST7735`, `TJpg_Decoder`, and `ArduinoJson`.
4. Locate lines 20, 21, and 22 in the code and replace the placeholders with your own network and API credentials:
   ```cpp
   const char* ssid = "YOUR_WIFI_SSID";
   const char* password = "YOUR_WIFI_PASSWORD";
   const char* groq_api_key = "YOUR_GROQ_API_KEY";
