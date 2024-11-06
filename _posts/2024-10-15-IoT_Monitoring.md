---
layout: post
title: Building a Real-time System Monitor with M5Stack and Firebase
categories: [IoT, ESP32, Python, Firebase]
---

In this article, I'll walk through how I built a real-time system monitoring solution using an M5Stack Core2 device, Firebase Realtime Database, and Python. This project creates a sleek dashboard that displays system metrics like CPU usage, memory utilization, and disk space, along with file entropy visualization and remote system actions.

## System Architecture

The project consists of three main components:
- M5Stack Core2 device for the display interface
- Python backend for system metrics collection
- Firebase Realtime Database for data synchronization

### Hardware Requirements
- M5Stack Core2 (ESP32-based device)
- WiFi connection
- Firebase account

### Software Stack
- ESP32 Arduino framework
- Firebase ESP Client library
- ArduinoJson for data parsing
- Python with psutil for system monitoring

## Part 1: Python Backend

Let's start with the metrics collection system. Here's the Python code that monitors system resources:

```python
import psutil
import time
import socket
import json

class SystemMonitor:
    def __init__(self):
        self.cpu_percent = 0
        self.memory_percent = 0
        self.disk_percent = 0

    def update_metrics(self):
        self.cpu_percent = psutil.cpu_percent()
        self.memory_percent = psutil.virtual_memory().percent
        self.disk_percent = psutil.disk_usage('/').percent

    def get_metrics(self):
        return {
            'cpu_percent': self.cpu_percent,
            'memory_percent': self.memory_percent,
            'disk_percent': self.disk_percent
        }
```

The Python backend uses `psutil` to collect system metrics and serves them over a socket connection. This allows for real-time monitoring of:
- CPU usage percentage
- Memory utilization
- Disk space consumption

## Part 2: M5Stack Interface

### Display System

The M5Stack interface is built with a multi-page design using an object-oriented approach. Here's how the metrics page is rendered:

```cpp
void drawMetricsPage() {
  M5.Lcd.fillScreen(BLACK);
  M5.Lcd.setCursor(10, 10);
  M5.Lcd.println("System Monitor");
  
  // Draw CPU bar
  M5.Lcd.drawRect(10, 40, 300, 30, WHITE);
  M5.Lcd.fillRect(12, 42, 296 * cpu_percent / 100.0, 26, RED);
  M5.Lcd.setCursor(10, 75);
  M5.Lcd.printf("CPU: %.1f%%", cpu_percent);
  
  // Draw Memory bar
  M5.Lcd.drawRect(10, 100, 300, 30, WHITE);
  M5.Lcd.fillRect(12, 102, 296 * memory_percent / 100.0, 26, GREEN);
  M5.Lcd.setCursor(10, 135);
  M5.Lcd.printf("Memory: %.1f%%", memory_percent);
  
  // Draw Disk bar
  M5.Lcd.drawRect(10, 160, 300, 30, WHITE);
  M5.Lcd.fillRect(12, 162, 296 * disk_percent / 100.0, 26, BLUE);
  M5.Lcd.setCursor(10, 195);
  M5.Lcd.printf("Disk: %.1f%%", disk_percent);
}
```

### Interactive Features

The interface includes touch navigation and multiple view modes:

```cpp
void handleTouch() {
  if (M5.Touch.ispressed()) {
    TouchPoint_t pos = M5.Touch.getPressPoint();
    if (pos.y > 220) {
      if (pos.x < 106) {
        currentPage = METRICS;
      } else if (pos.x < 212) {
        currentPage = ENTROPY;
      } else {
        currentPage = ACTIONS;
      }
      updateDisplay();
    } else if (currentPage == ACTIONS) {
      if (pos.y > 50 && pos.y < 90) {
        // Restart service button pressed
        Firebase.RTDB.setString(&fbdo, "/action", "restart_service");
      } else if (pos.y > 100 && pos.y < 140) {
        // Clear cache button pressed
        Firebase.RTDB.setString(&fbdo, "/action", "clear_cache");
      }
    }
  }
}
```

## Part 3: Firebase Integration

The project uses Firebase Realtime Database for real-time data synchronization. Here's how the Firebase connection is set up:

```cpp
// Firebase configuration
#define DATABASE_URL "YOUR_URL_HERE"

void setup() {
  // Configure Firebase
  config.database_url = DATABASE_URL;
  config.service_account.data.client_email = FIREBASE_CLIENT_EMAIL;
  config.service_account.data.project_id = FIREBASE_PROJECT_ID;
  config.service_account.data.private_key = FIREBASE_PRIVATE_KEY;

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
}
```

### Data Synchronization

The system updates metrics in real-time using Firebase:

```cpp
void updateMetrics() {
  if (Firebase.RTDB.getFloat(&fbdo, "/cpu_percent")) {
    cpu_percent = fbdo.floatData();
  }
  if (Firebase.RTDB.getFloat(&fbdo, "/memory_percent")) {
    memory_percent = fbdo.floatData();
  }
  if (Firebase.RTDB.getFloat(&fbdo, "/disk_percent")) {
    disk_percent = fbdo.floatData();
  }
  
  // Update entropy data
  if (Firebase.RTDB.getJSON(&fbdo, "/file_entropy")) {
    DynamicJsonDocument doc(1024);
    deserializeJson(doc, fbdo.jsonString());
    file_entropy.clear();
    for (JsonPair kv : doc.as<JsonObject>()) {
      file_entropy[kv.key().c_str()] = kv.value().as<float>();
    }
  }
}
```

## Part 4: Special Features

### Entropy Heat Map

One unique feature is the entropy visualization system:

```cpp
void drawEntropyPage() {
  M5.Lcd.fillScreen(BLACK);
  M5.Lcd.setCursor(10, 10);
  M5.Lcd.println("Entropy Heat Map");

  float min_entropy = 0;
  float max_entropy = 8;  // Maximum theoretical entropy for 8-bit data

  int y = 40;
  int height = 180 / file_entropy.size();
  for (const auto& entry : file_entropy) {
    float normalized_entropy = (entry.second - min_entropy) / (max_entropy - min_entropy);
    uint16_t color = M5.Lcd.color565(255 * normalized_entropy, 0, 255 * (1 - normalized_entropy));
    M5.Lcd.fillRect(10, y, 300, height, color);
    M5.Lcd.setCursor(320, y + height / 2 - 10);
    M5.Lcd.printf("%.2f", entry.second);
    y += height;
  }
}
```

### Remote Actions

The system supports remote actions through Firebase:

```cpp
void drawActionsPage() {
  M5.Lcd.fillScreen(BLACK);
  M5.Lcd.setCursor(10, 10);
  M5.Lcd.println("Remote Actions");

  M5.Lcd.fillRect(10, 50, 300, 40, BLUE);
  M5.Lcd.setCursor(20, 60);
  M5.Lcd.println("Restart Service");

  M5.Lcd.fillRect(10, 100, 300, 40, GREEN);
  M5.Lcd.setCursor(20, 110);
  M5.Lcd.println("Clear Cache");
}
```

## Project Setup

### PlatformIO Configuration

The project uses PlatformIO for building and dependency management:

```ini
[env:m5stack-core2]
platform = espressif32
board = m5stack-core2
framework = arduino
monitor_speed = 115200
lib_deps = 
    m5stack/M5Core2@^0.1.7
    bblanchon/ArduinoJson@^6.21.3
    mobizt/Firebase Arduino Client Library for ESP8266 and ESP32@^4.3.9
build_flags = 
    -DCORE_DEBUG_LEVEL=0
    -DBOARD_HAS_PSRAM
    -mfix-esp32-psram-cache-issue
```

## Key Technical Challenges

### 1. Real-time Data Updates

Ensuring smooth updates while maintaining responsive UI required careful timing:

```cpp
void loop() {
  if (Firebase.ready()) {
    updateMetrics();
    handleTouch();
  } else {
    // Handle connection errors
    M5.Lcd.fillScreen(BLACK);
    M5.Lcd.setCursor(10, 100);
    M5.Lcd.println("Firebase not ready");
    M5.Lcd.print("Error: ");
    M5.Lcd.println(fbdo.errorReason().c_str());
    delay(5000);
  }
  M5.update();
  delay(100);  // Prevent excessive updates
}
```

### 2. Memory Management

The project uses JSON parsing with size limitations:

```cpp
DynamicJsonDocument doc(1024);  // Allocate document on stack
deserializeJson(doc, fbdo.jsonString());
```

### 3. Error Handling

Robust error handling ensures system stability:

```cpp
if (!Firebase.ready()) {
    M5.Lcd.fillScreen(BLACK);
    M5.Lcd.setCursor(10, 100);
    M5.Lcd.println("Firebase not ready");
    M5.Lcd.print("Error: ");
    M5.Lcd.println(fbdo.errorReason().c_str());
    delay(5000);  // Wait before retry
}
```

## Future Improvements

Potential enhancements :
1. Historical data tracking and graphing
2. Additional system metrics
3. More sophisticated entropy analysis
4. Alert system for metric thresholds
5. Bluetooth connectivity option
6. Custom metric definitions

## Conclusion

This M5Stack system monitor project demonstrates the power of combining embedded systems with cloud services for real-time monitoring. The combo of Python backend, Firebase RTDB, and M5Stack's capabilities creates a powerful moinatinitoring solution.

The source code shows various best practices for:
- Real-time data visualization
- Touch interface design
- Cloud integration
- System monitoring
- Error handling

This project serves as a foundation for building more sophisticated monitoring and control systems using M5Stack devices, and maybe even one day allow for remote EDR in monitored computers...

## Resources

- [M5Stack Documentation](https://docs.m5stack.com/en/core/core2)
- [Firebase ESP32 Client](https://github.com/mobizt/Firebase-ESP32)
- [ArduinoJson Documentation](https://arduinojson.org/)
- [psutil Documentation](https://psutil.readthedocs.io/)


