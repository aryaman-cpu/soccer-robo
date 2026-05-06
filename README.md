# 🤖 4WD Bluetooth Sumo Robot Guide

## 📋 Component Checklist
* **Brain:** ESP32 Development Board
* **Muscles:** 2x BTS7960 Motor Driver Boards
* **Legs:** 4x Johnson DC Gear Motors (300 RPM) & 4x Rubber Tyres
* **Power:** Bonka 11.1V 3S LiPo Battery
* **Regulator:** Buck Converter (11.1V to 5V)
* **Controller:** Any generic "Bluetooth Serial Terminal" app on a phone

---

## 🛠️ Step-by-Step Wiring

### Power Management (Do this first!)
1. Connect the LiPo battery to the Buck Converter `IN+` and `IN-`.
2. Turn the screw on the converter until the `OUT` reads exactly 5.0V on a multimeter.
3. Disconnect the battery. Wire the converter's `OUT+` to the ESP32 `VIN` (or 5V) and `OUT-` to ESP32 `GND`.

### Drivetrain & Driver Power
1. Wire the positive terminals of both Left motors together, and negative terminals together. Connect them to `M+` and `M-` on the Left BTS7960.
2. Repeat the same parallel wiring for the Right motors and connect to the Right BTS7960.
3. Connect the `B+` and `B-` of both motor drivers directly to the LiPo battery.
4. **Crucial:** Connect the battery's negative terminal (Ground) to a `GND` pin on the ESP32.

### Logic Wiring (Matching the Code)
Connect the control pins from the drivers to the ESP32 as follows:

**Left Driver:**
* `R_EN` -> GPIO 27
* `L_EN` -> GPIO 14
* `RPWM` -> GPIO 25
* `LPWM` -> GPIO 26

**Right Driver:**
* `R_EN` -> GPIO 12
* `L_EN` -> GPIO 13
* `RPWM` -> GPIO 32
* `LPWM` -> GPIO 33

*Connect `VCC` on both drivers to ESP32 5V, and `GND` to ESP32 `GND`.*

---

## 💻 The Code
Upload this exact sketch to the ESP32. Connect to **"SUMO_BOT_OFFROAD"** using a Bluetooth Serial Terminal app and send standard characters (`F`, `B`, `L`, `R`, `S`) to drive.
```cpp
/*
SUMO ROBOT CONTROL SYSTEM
Targeted Hardware: ESP32 + Dual BTS7960
Description: Bluetooth-controlled tank drive with failsafe. 
*/
#include "BluetoothSerial.h"

// Check if Bluetooth is properly enabled 
#if !defined(CONFIG_BT_ENABLED) || !defined(CONFIG_BLUEDROID_ENABLED) 
#error Bluetooth is not enabled! Please run make menuconfig to enable it 
#endif

BluetoothSerial SerialBT;

// --- PIN ASSIGNMENTS --- 
// Left Side 
const int RPWM_L = 25; 
const int LPWM_L = 26; 
const int REN_L = 27; 
const int LEN_L = 14;

// Right Side 
const int RPWM_R = 32; 
const int LPWM_R = 33; 
const int REN_R = 12; 
const int LEN_R = 13;

// --- MOTOR SETTINGS --- 
const int freq = 15000;         // High frequency to avoid motor buzzing 
const int resolution = 8;       // 8-bit resolution (0-255) 
int baseSpeed = 255;            // Initial speed at max 
float turnFactor = 0.7;         // Slows down the inner track during turns

// --- SAFETY FAILSAFE --- 
unsigned long lastCommandTime = 0; 
const int timeout = 500;        // 500ms timeout if Bluetooth disconnects

void setup() { 
  Serial.begin(115200); 
  SerialBT.begin("SUMO_BOT_OFFROAD"); // Bluetooth Device Name 
  Serial.println("Robot Ready. Pair via Bluetooth.");

  // Initialize Enable Pins 
  pinMode(REN_L, OUTPUT); 
  pinMode(LEN_L, OUTPUT); 
  pinMode(REN_R, OUTPUT); 
  pinMode(LEN_R, OUTPUT);

  // Wake up the drivers 
  digitalWrite(REN_L, HIGH); 
  digitalWrite(LEN_L, HIGH); 
  digitalWrite(REN_R, HIGH); 
  digitalWrite(LEN_R, HIGH);

  // Configure PWM for ESP32 3.x 
  ledcAttach(RPWM_L, freq, resolution); 
  ledcAttach(LPWM_L, freq, resolution); 
  ledcAttach(RPWM_R, freq, resolution); 
  ledcAttach(LPWM_R, freq, resolution); 
}

// Low-level motor driving function 
void setMotorLeft(int speed) { 
  speed = constrain(speed, -255, 255); 
  if (speed > 0) { 
    ledcWrite(RPWM_L, speed); 
    ledcWrite(LPWM_L, 0); 
  } else if (speed < 0) { 
    ledcWrite(RPWM_L, 0); 
    ledcWrite(LPWM_L, -speed); 
  } else { 
    ledcWrite(RPWM_L, 0); 
    ledcWrite(LPWM_L, 0); 
  } 
}

void setMotorRight(int speed) { 
  speed = constrain(speed, -255, 255); 
  if (speed > 0) { 
    ledcWrite(RPWM_R, speed); 
    ledcWrite(LPWM_R, 0); 
  } else if (speed < 0) { 
    ledcWrite(RPWM_R, 0); 
    ledcWrite(LP
