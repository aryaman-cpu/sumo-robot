Here is the complete, updated Markdown guide. It now includes the electronics setup and your ESP32 Bluetooth control code, complete with explanations so your juniors understand exactly how the software interacts with the hardware. You can drop this straight into your GitHub repository's README.md.

🤖 Heavy-Duty 4WD Sumo Robot Build Guide
Welcome to the official build guide for our 4WD Sumo Robot. When building a competitive sumo bot—especially one with a heavy, protected tank-shaped chassis—standard motor drivers won't cut it. This guide will walk you through wiring an ESP32 brain with dual high-power BTS7960 motor drivers and uploading the Bluetooth control code to give our robot the massive pushing power it needs.

📋 Components List
1x ESP32 Microcontroller (The "brain")

1x ESP32 Expansion Board / Shield (For easy pin access and power distribution)

2x BTS7960 (IBT-2) Motor Drivers (High-current drivers, one for the left side, one for the right)

4x Johnson DC Motors (High torque for pushing)

1x 11.1V Bonka LiPo Battery (Logic battery)

1x High-Capacity Motor Battery (12V–14.8V equivalent)

1x DC Barrel Jack Connector

Jumper wires & thick gauge wire (for power lines)

⚡ Step 1: Understanding Dual-Power Isolation
One of the most important rules in robotics is isolating your power supplies. Motors draw massive amounts of current, especially when pushing against an opponent. If the motors and the microcontroller share the same battery, a sudden current draw can cause a voltage drop, forcing the ESP32 to restart mid-battle.

To prevent this, we use a two-battery system:

The Brain Power: The 11.1V Bonka battery connects to the barrel jack, which plugs directly into the ESP32 Expansion Board. The board safely steps this down to power the ESP32 logic.

The Muscle Power: The secondary, high-capacity battery connects only to the thick power terminals on the motor drivers.

⚙️ Step 2: Wiring the Motors (Tank Steering)
We are using a skid-steer (or tank) setup. This means the two left motors act together as one unit, and the two right motors act together as another.

Left Side Drive: Take the two motors on the left side of the chassis. Wire their positive terminals together, and their negative terminals together (parallel). Connect these to the thick Motor Output screw terminals (M+ and M-) on the Left BTS7960.

Right Side Drive: Do the exact same thing for the two right motors, connecting them to the Right BTS7960.

Motor Power: Connect the positive and negative wires from your high-capacity motor battery to the thick VCC and GND screw terminals on both BTS7960 drivers. Use thick wire for this step to handle the high current!

🧠 Step 3: Logic Connections (ESP32 to BTS7960)
The BTS7960 uses a set of pins to control forward speed, reverse speed, and to enable the chip. Both BTS7960 boards need 5V and Ground from the ESP32 to power their internal logic chips. Connect VCC and GND on the BTS7960 header block to 5V and GND on the ESP32 Expansion Board.

Left Motor Driver:

RPWM to GPIO 25

LPWM to GPIO 26

R_EN to GPIO 27

L_EN to GPIO 14

Right Motor Driver:

RPWM to GPIO 32

LPWM to GPIO 33

R_EN to GPIO 12

L_EN to GPIO 13

💻 Step 4: The Control Code
This code turns the ESP32 into a Bluetooth receiver that translates commands from a smartphone controller app into PWM (Pulse Width Modulation) signals for the motor drivers.

Note for builders: This code uses the updated ESP32 Arduino Core 3.x syntax for PWM (ledcAttach). Ensure your Arduino IDE board manager is up to date.

C++
// ================= Bluetooth =================
#include "BluetoothSerial.h"
BluetoothSerial SerialBT;

// ================= LEFT SIDE =================
#define RPWM_L 25
#define LPWM_L 26
#define REN_L  27
#define LEN_L  14

// ================= RIGHT SIDE =================
#define RPWM_R 32
#define LPWM_R 33
#define REN_R  12
#define LEN_R  13

// ================= PWM =================
const int freq = 15000;
const int resolution = 8;

// ================= SPEED =================
int baseSpeed = 255;
float turnFactor = 0.7;

// ================= FAILSAFE =================
unsigned long lastCommandTime = 0;
const int timeout = 500;

// ================= SETUP =================
void setup() {
  Serial.begin(115200);
  SerialBT.begin("OFFROAD CAR");

  pinMode(REN_L, OUTPUT);
  pinMode(LEN_L, OUTPUT);
  pinMode(REN_R, OUTPUT);
  pinMode(LEN_R, OUTPUT);

  digitalWrite(REN_L, HIGH);
  digitalWrite(LEN_L, HIGH);
  digitalWrite(REN_R, HIGH);
  digitalWrite(LEN_R, HIGH);

  // ESP32 3.x PWM
  ledcAttach(RPWM_L, freq, resolution);
  ledcAttach(LPWM_L, freq, resolution);
  ledcAttach(RPWM_R, freq, resolution);
  ledcAttach(LPWM_R, freq, resolution);
}

// ================= MOTOR CONTROL =================
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

// ================= LOOP =================
void loop() {

  if (SerialBT.available()) {
    char cmd = SerialBT.read();
    lastCommandTime = millis();

    switch (cmd) {

      // ===== STOP (instant) =====
      case 'S': 
        move(0, 0); 
        break;

      // ===== Forward =====
      case 'F': move(baseSpeed, baseSpeed); break;

      // ===== Reverse =====
      case 'B': move(-baseSpeed, -baseSpeed); break;

      // ===== Forward Turns =====
      case 'G': move(baseSpeed * turnFactor, baseSpeed); break;
      case 'I': move(baseSpeed, baseSpeed * turnFactor); break;

      // ===== Reverse Turns =====
      case 'J': move(-baseSpeed * turnFactor, -baseSpeed); break;
      case 'H': move(-baseSpeed, -baseSpeed * turnFactor); break;

      // ===== Spin =====
      case 'L': move(-baseSpeed, baseSpeed); break;
      case 'R': move(baseSpeed, -baseSpeed); break;

      // ===== Speed Control =====
      case '0': baseSpeed = 0; break;
      case '2': baseSpeed = 80; break;
      case '4': baseSpeed = 120; break;
      case '6': baseSpeed = 160; break;
      case '8': baseSpeed = 200; break;
      case 'q': baseSpeed = 255; break;

    }
  }

  // ===== FAILSAFE (backup only) =====
  if (millis() - lastCommandTime > timeout) {
    move(0, 0);
  }
}
🔍 Code Highlights for Juniors:
Enable Pins (REN / LEN): In the setup() function, we pull all the enable pins HIGH. This wakes up the BTS7960 drivers so they are ready to accept speed commands.

The Failsafe: Bluetooth connections can drop. The lastCommandTime variable tracks the last time the ESP32 received a command. If 500 milliseconds pass with no new commands (the timeout), the loop automatically forces the motors to stop (move(0, 0);). This prevents a runaway robot!

Tank Turning: Notice how the Spin commands (L and R) send a positive speed to one side and a negative speed to the other. This drives the tracks in opposite directions, allowing the bot to turn perfectly on its own axis.

🛑 Final Assembly & Safety Notes
LiPo Battery Warning: LiPo batteries can be dangerous if short-circuited. Never let the red and black wires of your batteries touch. Always double-check your polarity before plugging anything in.

Chassis Protection: When securing the electronics and wiring inside the 40x40x30 trapezium shell, make sure nothing is loose. The components need to be completely protected inside the chassis so they aren't damaged when taking hits during a sumo match.

Common Grounds: Ensure the ground line from the ESP32 is securely connected to the ground pin on the BTS7960 logic header. Without a common ground, the PWM signals won't be read correctly.