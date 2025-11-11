# my-electromagnetic-hand
Arduino code for an electromagnetic robotic hand project
/* electromagnetic_hand.ino

Example Arduino sketch for an "electromagnetic hand" project.

FEATURES (provided by this example):

Controls an electromagnet (solenoid) using PWM through an N-channel MOSFET

Controls 3 finger servos (can be expanded) for opening/closing the hand

Reads a potentiometer to vary electromagnet strength (PWM duty)

Optional push-button for quick on/off toggle

Optional Bluetooth command interface (via Serial) for remote control


ASSUMPTIONS / SAFETY NOTES

Electromagnet draws higher current than Arduino pins can supply. Use an external power supply for the electromagnet (e.g., 12V) and an N-channel logic-level MOSFET (e.g., IRLZ44N, IRLZ34) to switch the ground side. Place a diode across the coil if it's a solenoid.

Use a common ground between Arduino and the electromagnet power supply.

Add a flyback diode (if coil is not integrated with diode) and a TVS or snubber if needed.

Check current ratings; add a current-limiting resistor or a PWM frequency change if coil heating is a concern. Do not drive continuously at maximum duty cycle for long periods without thermal management.


HARDWARE LIST (example)

Arduino Uno / Nano

MOSFET (N-channel, logic-level)

Electromagnet or solenoid (specify voltage/current)

External power supply for the electromagnet (e.g., 12V 5A)

Servos (SG90 or larger, 3 used here)

Potentiometer (10k) for manual strength control

Push-button (momentary) for toggling

(Optional) HC-05 Bluetooth module for Serial commands


WIRING (example)

Electromagnet + -> External PSU + (e.g., +12V)

Electromagnet - -> MOSFET Drain

MOSFET Source -> PSU GND (and Arduino GND must be connected to PSU GND)

MOSFET Gate -> Arduino digital PWM pin (through 220 ohm resistor recommended)

Flyback diode across coil: diode cathode to +12V, anode to MOSFET Drain (if coil doesn't include diode)

Servos -> External 5V supply (or Arduino 5V if current small) ; signal wires -> Arduino PWM pins

Potentiometer middle pin -> A0, ends to 5V and GND

Button -> Arduino digital input with pullup (to GND when pressed)

HC-05 TX->RX, RX->TX (via voltage divider for RX if module expects 3.3V)


QUICK NOTES ON USAGE

Potentiometer controls PWM strength (0..255) for the electromagnet

Button toggles electromagnet on/off (remember: PWM still obeys pot value)

Bluetooth/Serial commands: 'ON', 'OFF', 'SET x' (x = 0..255), 'OPEN', 'CLOSE'


*/

#include <Servo.h>

// -------------------- USER-CONFIGURABLE PINS -------------------- const int MAG_PWM_PIN = 9;       // PWM pin driving MOSFET gate (must be PWM-capable) const int POT_PIN     = A0;      // Potentiometer analog input for strength (0..1023) const int BUTTON_PIN  = 2;       // Momentary push-button to toggle on/off

// Servos: change pins / counts as needed const int SERVO_COUNT = 3; const int SERVO_PINS[SERVO_COUNT] = {5, 6, 10}; // example PWM pins for servos (10 is PWM on Uno)

// -------------------- CONFIGURATION -------------------- const bool USE_BLUETOOTH = true; // set false if not using Serial remote

// Servo positions (change per your mechanical build) const int SERVO_OPEN_POS  = 10;  // degrees const int SERVO_CLOSED_POS = 90; // degrees

// Safety limits const int MAX_PWM = 255;         // maximum PWM value const int DEFAULT_PWM = 0;       // start off

// Debounce const unsigned long BUTTON_DEBOUNCE_MS = 50;

// -------------------- GLOBALS -------------------- Servo servos[SERVO_COUNT]; int currentPWM = DEFAULT_PWM;   // current PWM value (0..255) bool magnetOn = false;         // logical state (on/off via button or command) unsigned long lastButtonChange = 0; int lastButtonState = HIGH;     // using INPUT_PULLUP

void setup() { // Serial for debugging / bluetooth commands Serial.begin(9600); if (USE_BLUETOOTH) Serial.println("Electromagnetic hand starting...");

// pin modes pinMode(MAG_PWM_PIN, OUTPUT); digitalWrite(MAG_PWM_PIN, LOW);

pinMode(BUTTON_PIN, INPUT_PULLUP); // button to ground when pressed

// Attach servos for (int i = 0; i < SERVO_COUNT; ++i) { servos[i].attach(SERVO_PINS[i]); servos[i].write(SERVO_OPEN_POS); }

// Initialize magnet off analogWrite(MAG_PWM_PIN, 0); }

void loop() { // Read potentiometer and map to PWM int pot = analogRead(POT_PIN); // 0..1023 int potPWM = map(pot, 0, 1023, 0, MAX_PWM);

// Button handling with debounce (toggle) int rawButton = digitalRead(BUTTON_PIN); if (rawButton != lastButtonState) { lastButtonChange = millis(); lastButtonState = rawButton; } if ((millis() - lastButtonChange) > BUTTON_DEBOUNCE_MS) { if (rawButton == LOW) { // pressed // toggle magnetOn = !magnetOn; // quick visual debug Serial.print("Button toggled, magnetOn="); Serial.println(magnetOn); // simple debounce wait (prevent rapid toggles) delay(150); } }

// If magnet is on, follow potentiometer; otherwise 0 if (magnetOn) { currentPWM = potPWM; } else { currentPWM = 0; }

// Apply PWM to MOSFET (electromagnet) analogWrite(MAG_PWM_PIN, currentPWM);

// Optional: automatic gripping behavior when magnet is high // Example: if magnet strong enough, close fingers if (currentPWM > 150) { setFingersClosed(); } else { setFingersOpen(); }

// Handle Serial / Bluetooth commands if (USE_BLUETOOTH && Serial.available()) { String cmd = Serial.readStringUntil('\n'); cmd.trim(); cmd.toUpperCase(); handleCommand(cmd); }

// Small loop delay delay(30); }

// -------------------- Helper functions --------------------

void setFingersClosed() { for (int i = 0; i < SERVO_COUNT; ++i) { servos[i].write(SERVO_CLOSED_POS); } }

void setFingersOpen() { for (int i = 0; i < SERVO_COUNT; ++i) { servos[i].write(SERVO_OPEN_POS); } }

void handleCommand(const String &cmd) { // Commands: ON, OFF, SET <0..255>, OPEN, CLOSE Serial.print("CMD: "); Serial.println(cmd); if (cmd == "ON") { magnetOn = true; Serial.println("Magnet ON (logical)"); } else if (cmd == "OFF") { magnetOn = false; analogWrite(MAG_PWM_PIN, 0); Serial.println("Magnet OFF (logical)"); } else if (cmd.startsWith("SET ")) { // parse value int val = cmd.substring(4).toInt(); val = constrain(val, 0, MAX_PWM); currentPWM = val; if (val > 0) magnetOn = true; else magnetOn = false; analogWrite(MAG_PWM_PIN, currentPWM); Serial.print("SET PWM="); Serial.println(val); } else if (cmd == "OPEN") { setFingersOpen(); Serial.println("Fingers OPEN"); } else if (cmd == "CLOSE") { setFingersClosed(); Serial.println("Fingers CLOSED"); } else { Serial.println("Unknown command"); } }

// -------------------- END --------------------
