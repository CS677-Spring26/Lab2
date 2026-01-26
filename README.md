# Lab 2: LLM Gesture Monitor

## Background 

In Lab 1, you mastered local hardware control. Lab 2 shifts the focus to **Distributed IoT Systems**. You will transform an ESP32 into an "Intelligent Edge" device that captures raw movement and delegates the "thinking" to a Large Language Model (LLM) on a seperate server.

You will physically move your sensor to perform distinct gestures such as drawing a circle in the air, a sharp zigzag, or a rapid "shake" motion. As you move the device, the MPU-6050 captures the resulting changes in angular velocity and acceleration across its three axes. Instead of writing rigid, traditional algorithms to detect these patterns, you will stream this raw time-series data to a Gemini-powered server. The Large Language Model will "read" the stream of numbers, recognize the underlying physical intent behind the fluctuations, and determine exactly which movement was performed.

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

## Task 2: ESP32 High-Speed Sampling

The ESP32 acts as an **HTTP Client**. It captures data and "pushes" it to the internet.

#### **Sampling Logic**

To detect a gesture (like a circle), you need a "window" of data.

* **Window:** 2 Seconds of continuous movement.
* **Interval:** 50ms (Total: 40 samples).
* **Packaging:** The samples are stored in a single JSON array string.

#### **Code Structure (`HTTP POST`)**

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

const char* proxyUrl = "https://your-ngrok-url.ngrok-free.app/classify";

void loop() {
  // 1. Read MPU-6050 40 times (Gyro X, Y, Z).
  // 2. Format into: [{"x":0.1, "y":0.2}, {"x":0.12, "y":0.21}, ...]
  // 3. Send POST request:
  HTTPClient http;
  http.begin(proxyUrl);
  http.addHeader("Content-Type", "application/json");
  int httpResponseCode = http.POST(jsonPayload);
  
  if (httpResponseCode > 0) {
    Serial.println("AI Result: " + http.getString());
  }
  http.end();
  delay(5000); // 5-second pause
}

```
---

## Task 3: The Gemini AI Proxy Server

You will write a Flask server (`app.py`) that acts as the bridge between your ESP32 and the Gemini's Servers. This server receives the sensor data, consults Gemini, and sends the final answer back to the microcontroller.

#### **1. Create the Flask HTTP Endpoint**

Define a route to receive the data and return the AI's response directly to the ESP32.

```python
@app.route('/classify', methods=['POST'])
def classify_gesture():
    sensor_data = request.json  # Extract JSON array from ESP32
    # ... (AI Logic below) ...
    return response.text, 200   # Sends the classification back to the ESP32

```

#### **2. Configure Structured JSON Output**

Use Gemini's JSON mode to ensure the AI returns a parseable object instead of a long sentence.

```python
generation_config = {"temperature": 0.1, "response_mime_type": "application/json"}
model = genai.GenerativeModel("gemini-1.5-flash", generation_config=generation_config)

```

#### **3. Define the Classification Prompt**

Instruct the AI to "read" the numbers and categorize the movement into specific labels. You may change or modify these labels depending on whether the LLM is successful in classification or not.

```python
prompt = f"YOUR PROMPT HERE. Return JSON: {{'gesture': 'RESULT'}}. Data: {sensor_data}"

```

#### **4. Execute and Print Results**

Call the model and send the final result to your ESP32-S3 as a response. The ESP32-S3 should print the gesture on its serial monitor.

```python
response = model.generate_content(prompt)
print(f"Gemini Classification: {response.text}")

```

#### **5. Running the Proxy Server**
Start the Server: Execute your script using python app.py and ensure the terminal displays **Running on http://0.0.0.0:5000** before performing any gestures.


### **System Verification**

Once both the Python script and the ESP32 code are running, you should see the following flow:

1. **Move:** You rotate or zigzag the MPU-6050 for 5 seconds.
2. **Push:** The ESP32 Serial Monitor says `Sending Data...`.
3. **Think:** Your Python terminal prints `Gemini Classification: {"gesture": "Clockwise"}`.
4. **Result:** The ESP32 receives that text and prints `AI Result: {"gesture": "Clockwise"}` to its own Serial Monitor.
---
