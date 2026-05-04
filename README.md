# 🤖 Project: Heavyweight 4WD Bluetooth Sumo Robot
## Technical Manual & Assembly Guide v1.0

This document provides a comprehensive breakdown of the hardware, software, and mechanical integration for a high-torque Sumo Robot. This build is designed for maximum pushing power, utilizing high-current motor drivers and an ESP32 microcontroller for real-time Bluetooth control.

---

## 📑 Table of Contents
1. [Executive Summary](#executive-summary)
2. [Component Specifications](#component-specifications)
3. [System Architecture](#system-architecture)
4. [Hardware Wiring Guide](#hardware-wiring-guide)
5. [The Brain: ESP32 & Bluetooth Logic](#the-brain-esp32--bluetooth-logic)
6. [The Muscle: BTS7960 Driver Theory](#the-muscle-bts7960-driver-theory)
7. [The Code: Full Implementation](#the-code-full-implementation)
8. [Mechanical Integration & Chassis](#mechanical-integration--chassis)
9. [Safety & Maintenance](#safety--maintenance)
10. [Troubleshooting FAQ](#troubleshooting-faq)

---

## 1. Executive Summary
In Robot Sumo, the objective is simple: push the opponent out of the ring. To achieve this, a robot needs two things: **Mass** and **Torque**. This build utilizes four Johnson motors in a "tank-drive" configuration, controlled by an ESP32. By using dual BTS7960 drivers, we can handle currents up to 43A per side, ensuring the motors never stall during a head-on collision.

---

## 2. Component Specifications

| Component | Role | Key Specification |
| :--- | :--- | :--- |
| **ESP32 WROOM** | Microcontroller | Dual-core, Integrated Bluetooth, High-speed PWM |
| **BTS7960 (IBT-2)** | Motor Driver | 43A Max Current, H-Bridge configuration |
| **Johnson Motors** | Actuators | High torque, Metal geared DC motors |
| **Bonka 11.1V** | Logic Power | 3-cell LiPo for stable ESP32 performance |
| **High-Cap Battery** | Motor Power | 12V-14.8V for maximum motor RPM and torque |
| **Expansion Board**| Connectivity | Easy access to GPIOs and stable power rails |

---

## 3. System Architecture



The system is split into two isolated loops:
1.  **The Logic Loop:** Powered by the Bonka 11.1V battery via the barrel jack. This ensures the ESP32 doesn't reset when the motors draw massive current.
2.  **The Power Loop:** Powered by the primary motor battery. This flows directly into the BTS7960 drivers and then to the Johnson motors.

**Note:** Always ensure the Ground (GND) of the ESP32 is connected to the GND of the motor drivers to create a common reference for the PWM signals.

---

## 4. Hardware Wiring Guide

Refer to the provided file **circuit_image (3).jpg** for the master schematic.

### A. Power Distribution
*   **Logic:** Plug the Bonka 11.1V LiPo into the **DC Barrel Jack** of the ESP32 Expansion Board.
*   **Motors:** Connect the High-Capacity battery positive (+) to the `VCC` and negative (-) to the `GND` screw terminals on both BTS7960 drivers.

### B. Motor Connections
Connect the 4 Johnson motors in a "Skid Steer" (Tank) arrangement:
*   **Left Side:** Pair the two left motors. Connect their wires to the `M+` and `M-` terminals of the **Left BTS7960**.
*   **Right Side:** Pair the two right motors. Connect their wires to the `M+` and `M-` terminals of the **Right BTS7960**.

### C. ESP32 to BTS7960 (Logic Headers)
Each driver has an 8-pin logic header. Wire them as follows:

**Left Driver:**
*   `VCC` -> 5V (Expansion Board)
*   `GND` -> GND (Expansion Board)
*   `R_EN` -> **GPIO 27**
*   `L_EN` -> **GPIO 14**
*   `RPWM` -> **GPIO 25**
*   `LPWM` -> **GPIO 26**

**Right Driver:**
*   `VCC` -> 5V (Expansion Board)
*   `GND` -> GND (Expansion Board)
*   `R_EN` -> **GPIO 12**
*   **L_EN** -> **GPIO 13**
*   `RPWM` -> **GPIO 32**
*   `LPWM` -> **GPIO 33**

---

## 5. The Brain: ESP32 & Bluetooth Logic



The ESP32 is chosen for its superior speed and Bluetooth Serial capabilities. In this setup, the ESP32 acts as a transparent bridge between your smartphone and the hardware.
*   **Bluetooth Serial:** We use the `BluetoothSerial.h` library to create a virtual serial port over the air.
*   **PWM Control:** The motors are controlled via Pulse Width Modulation. By rapidly switching the power on and off, we can control the speed of the Johnson motors from 0% to 100%.

---

## 6. The Muscle: BTS7960 Driver Theory
The BTS7960 is a fully integrated high-current H-bridge. 
*   **R_EN & L_EN:** These are the Enable pins. When they are HIGH, the driver is active.
*   **RPWM & LPWM:** These control the direction. 
    *   PWM on `RPWM` + 0 on `LPWM` = Forward.
    *   0 on `RPWM` + PWM on `LPWM` = Reverse.
    *   This dual-PWM control allows for extremely smooth acceleration curves.

---

## 7. The Code: Full Implementation

Copy and paste this code into the Arduino IDE. Ensure you have the ESP32 board package installed.
```cpp
/*
 * SUMO ROBOT CONTROL SYSTEM
 * Targeted Hardware: ESP32 + Dual BTS7960
 * Description: Bluetooth-controlled tank drive with failsafe.
 */

#include "BluetoothSerial.h"

// Check if Bluetooth is properly enabled
#if !defined(CONFIG_BT_ENABLED) || !defined(CONFIG_BLUEDROID_ENABLED)
#error Bluetooth is not enabled! Please run `make menuconfig` to enable it
#endif

BluetoothSerial SerialBT;

// --- PIN ASSIGNMENTS ---
// Left Side
const int RPWM_L = 25; 
const int LPWM_L = 26; 
const int REN_L  = 27; 
const int LEN_L  = 14; 

// Right Side
const int RPWM_R = 32; 
const int LPWM_R = 33; 
const int REN_R  = 12; 
const int LEN_R  = 13; 

// --- MOTOR SETTINGS ---
const int freq = 15000;    // High frequency to avoid motor buzzing
const int resolution = 8;  // 8-bit resolution (0-255)
int baseSpeed = 255;       // Initial speed at max
float turnFactor = 0.7;    // Slows down the inner track during turns

// --- SAFETY FAILSAFE ---
unsigned long lastCommandTime = 0;
const int timeout = 500;   // 500ms timeout if Bluetooth disconnects

void setup() {
  Serial.begin(115200);
  SerialBT.begin("SUMO_BOT_OFFROAD"); // Bluetooth Device Name
  Serial.println("Robot Ready. Pair via Bluetooth.");

  // Initialize Enable Pins
  pinMode(REN_L, OUTPUT); pinMode(LEN_L, OUTPUT);
  pinMode(REN_R, OUTPUT); pinMode(LEN_R, OUTPUT);

  // Wake up the drivers
  digitalWrite(REN_L, HIGH); digitalWrite(LEN_L, HIGH);
  digitalWrite(REN_R, HIGH); digitalWrite(LEN_R, HIGH);

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
    ledcWrite(LPWM_R, -speed);
  } else {
    ledcWrite(RPWM_R, 0);
    ledcWrite(LPWM_R, 0);
  }
}

void move(int leftSpeed, int rightSpeed) {
  setMotorLeft(leftSpeed);
  setMotorRight(rightSpeed);
}

void loop() {
  // Check for incoming Bluetooth data
  if (SerialBT.available()) {
    char cmd = SerialBT.read();
    lastCommandTime = millis(); // Reset failsafe timer

    switch (cmd) {
      case 'F': move(baseSpeed, baseSpeed); break; // Forward
      case 'B': move(-baseSpeed, -baseSpeed); break; // Back
      case 'L': move(-baseSpeed, baseSpeed); break; // Spin Left
      case 'R': move(baseSpeed, -baseSpeed); break; // Spin Right
      case 'G': move(baseSpeed * turnFactor, baseSpeed); break; // Forward-Left
      case 'I': move(baseSpeed, baseSpeed * turnFactor); break; // Forward-Right
      case 'S': move(0, 0); break; // Stop
      
      // Speed Step Control
      case '0': baseSpeed = 0; break;
      case '4': baseSpeed = 120; break;
      case '8': baseSpeed = 200; break;
      case 'q': baseSpeed = 255; break;
    }
  }

  // FAILSAFE: If no command received in 500ms, stop the robot
  if (millis() - lastCommandTime > timeout) {
    move(0, 0);
  }
}