# Lab 2: LLM Health Monitor

## Background

In Lab 1, you mastered local hardware control. Lab 2 shifts the focus to **Distributed IoT Systems**. You will transform an ESP32 into an "Intelligent Health" device that captures biometric data and delegates the "thinking" to a Large Language Model (LLM) on a separate server.

You will use a MAX30102 Pulse Oximeter to measure Heart Rate (BPM) and Blood Oxygen (SpO2). The sensor captures photoplethysmogram (PPG) data by shining Red and IR light through your finger. Instead of relying on rigid, pre-programmed thresholds to determine if a user is "healthy," you will stream these calculated vitals to a Gemini-powered server. The Large Language Model will "read" the biometrics, understand the context, and provide actionable medical recommendations or wellness advice.

---

## Task 1: The Local Gateway (Mobile Hotspot)

Your ESP32 and Laptop must be on the same "local wifi" to communicate. Because campus Wi-Fi blocks device-to-device talking, you will use a **Mobile Hotspot**. **Both** your Laptop and ESP32 must connect to the same phone hotspot. This puts them behind a single gateway.

#### **ESP32 Wi-Fi Code**

Add this to your sketch to join the network:

```cpp
#include <WiFi.h>

const char* ssid = "YOUR_HOTSPOT_NAME"; 
const char* password = "YOUR_PASSWORD";

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) { 
    delay(500); Serial.print("."); 
  }
  Serial.println("\nConnected!");
  Serial.println(WiFi.localIP()); 
}

```

#### **The Ngrok Tunnel**

1. **Download & Auth:** Ensure you have signed up at [ngrok.com](https://dashboard.ngrok.com/signup) and added your authtoken to your PC's terminal. (You are recommended to use choco if you are on Windows).
2. **Start the Tunnel:** Run the following command in your terminal:
`ngrok http 5000`
3. **Copy the URL:** Look for the **Forwarding** line (e.g., `https://1234-abcd.ngrok-free.app`).
* This is the "Public Address" of your laptop.
* Any data the ESP32 sends to this URL will be forwarded to your Flask app running on port 5000 on your PC.



---

## Task 2: MAX30102 Sensor Setup & Sampling

The ESP32 acts as an **HTTP Client**. It processes biometric data locally and "pushes" the results to the internet.

#### **Hardware & Library Configuration**

1. **Library Setup:** In the Arduino IDE, go to **Sketch > Include Library > Manage Libraries**. Search for and install **"SparkFun MAX3010x Pulse and Proximity Sensor Library"**. This library handles the low-level register communication.
2. **I2C Pin Configuration:** The MAX30102 communicates via I2C (SDA/SCL). On the ESP32-S3, you must explicitly define which pins are used for I2C in your code, or the sensor will not be found.
3. **Sensor Placement & Sensitivity:**
* **Placement:** The sensor must be placed firmly against the fleshy part of your finger.
* **Light Sensitivity:** The sensor measures light absorption. **Ambient light (sunlight or bright room lights) will corrupt the data.** Cover the sensor and your finger with a dark cloth or your other hand while reading.
* **Stability:** Keep your hand still. Motion creates noise that ruins the BPM calculation.



#### **Sampling Logic & Calculations**

To get an accurate reading, we cannot use a single instantaneous sample. We must average data over time.

* **Buffer Length:** The standard algorithm requires a buffer of roughly **100 samples** (approx. 4 seconds of data) to detect peaks (heartbeats) and calculate the ratio of Red to IR light absorption (SpO2).
* **Minimum Calculations:** You do not need to write the DSP math from scratch. The library includes a file `spo2_algorithm.h`. You will pass the raw Red and IR buffers to the function `maxim_heart_rate_and_oxygen_saturation()`, which computes the valid SpO2 and BPM for you.

#### **Code Structure (`HTTP POST`)**

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <Wire.h>
#include "MAX30105.h"
#include "spo2_algorithm.h" // Ensure this file is in your project folder

MAX30105 particleSensor;

// Pin Definitions for ESP32-S3 (Check your specific board pinout!)
#define I2C_SDA 42 
#define I2C_SCL 41

// Buffer setup
#define MAX_BRIGHTNESS 255
uint32_t irBuffer[100]; // Infrared LED sensor data
uint32_t redBuffer[100];  // Red LED sensor data
int32_t bufferLength = 100; // Data length
int32_t spo2; // Calculated SpO2
int8_t validSPO2; // Indicator to show if value is valid
int32_t heartRate; // Calculated heart rate
int8_t validHeartRate; // Indicator to show if value is valid

const char* proxyUrl = "https://your-ngrok-url.ngrok-free.app/recommend";

void setup() {
  Serial.begin(115200);
  // ... (WiFi setup from Task 1) ...

  // Initialize I2C with specific pins
  Wire.begin(I2C_SDA, I2C_SCL);

  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30102 was not found. Please check wiring/power.");
    while (1);
  }

  // Sensor Configuration
  byte ledBrightness = 60; // Options: 0=Off to 255=50mA
  byte sampleAverage = 4; // Options: 1, 2, 4, 8, 16, 32
  byte ledMode = 2; // Options: 1 = Red only, 2 = Red + IR, 3 = Red + IR + Green
  byte sampleRate = 100; // Options: 50, 100, 200, 400, 800, 1000, 1600, 3200
  int pulseWidth = 411; // Options: 69, 118, 215, 411
  int adcRange = 4096; // Options: 2048, 4096, 8192, 16384

  particleSensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange);
}

void loop() {
  // 1. Collect 100 samples (approx 4 seconds)
  for (byte i = 0 ; i < bufferLength ; i++) {
    while (particleSensor.available() == false) 
      particleSensor.check(); // Check the sensor for new data

    redBuffer[i] = particleSensor.getRed();
    irBuffer[i] = particleSensor.getIR();
    particleSensor.nextSample(); // We're finished with this sample so move to next sample
  }

  // 2. Calculate SpO2 and Heartrate using library algorithm
  maxim_heart_rate_and_oxygen_saturation(irBuffer, bufferLength, redBuffer, &spo2, &validSPO2, &heartRate, &validHeartRate);

  // 3. Only send if data is valid
  if (validSPO2 == 1 && validHeartRate == 1) {
    String jsonPayload = "{\"spo2\": " + String(spo2) + ", \"bpm\": " + String(heartRate) + "}";
    
    HTTPClient http;
    http.begin(proxyUrl);
    http.addHeader("Content-Type", "application/json");
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode > 0) {
      Serial.println("AI Advice: " + http.getString());
    }
    http.end();
  } else {
    Serial.println("Reading failed. Please adjust finger position.");
  }
}

```

---

## Task 3: The Gemini AI Proxy Server

You will write a Flask server (`app.py`) that acts as the bridge between your ESP32 and the Gemini Servers. This server receives the SpO2 and BPM data, consults Gemini, and sends medical recommendations back to the microcontroller.

#### **1. Create the Flask HTTP Endpoint**

Define a route to receive the data and return the AI's response directly to the ESP32.

```python
@app.route('/recommend', methods=['POST'])
def recommend_health():
    sensor_data = request.json  # Extract JSON (spo2 and bpm) from ESP32
    # ... (AI Logic below) ...
    return response.text, 200   # Sends the recommendation back to the ESP32

```

#### **2. Configure Structured JSON Output**

Use Gemini's JSON mode to ensure the AI returns a parseable object instead of a long sentence.

```python
generation_config = {"temperature": 0.1, "response_mime_type": "application/json"}
model = genai.GenerativeModel("gemini-1.5-flash", generation_config=generation_config)

```

#### **3. Define the Medical Prompt**

Instruct the AI to evaluate the vitals and provide a short, actionable recommendation.

```python
prompt = f"Analyze these vitals. Return JSON: {{'status': 'HEALTH_STATUS', 'recommendation': 'SHORT_ADVICE'}}. Data: {sensor_data}"

```

#### **4. Execute and Print Results**

Call the model and send the final result to your ESP32-S3 as a response. The ESP32-S3 should print the advice on its serial monitor.

```python
response = model.generate_content(prompt)
print(f"Gemini Recommendation: {response.text}")

```

#### **5. Running the Proxy Server**

Start the Server: Execute your script using `python app.py` and ensure the terminal displays **Running on [http://0.0.0.0:5000**](http://0.0.0.0:5000) before placing your finger on the sensor.

### **System Verification**

Once both the Python script and the ESP32 code are running, you should see the following flow:

1. **Sense:** You place your finger on the MAX30102 for ~4 seconds.
2. **Push:** The ESP32 calculates SpO2/BPM and sends the data (e.g., `{"spo2": 98, "bpm": 75}`).
3. **Think:** Your Python terminal prints `Gemini Recommendation: {"status": "Normal", "recommendation": "Vitals look good. Maintain hydration."}`.
4. **Result:** The ESP32 receives that text and prints `AI Advice: ...` to its own Serial Monitor.

---
