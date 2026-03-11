#include <Wire.h>

// Pins for ESP32
const int RELAY_IGN = 25;
const int RELAY_START = 26;
const int RELAY_HORN = 27;
const int RELAY_HEAD = 14;

const int RF_BTN1 = 32;
const int RF_BTN2 = 33;
const int RF_BTN3 = 34;
const int RF_BTN4 = 35;

const int MPU_ADDR = 0x68;
const int MPU_SDA = 21;
const int MPU_SCL = 22;

bool armed = false;
bool relayHeadlightOn = false;

float lastAccel = 0;
unsigned long lastMotionTime = 0;

void setup() {
  pinMode(RELAY_IGN, OUTPUT);     digitalWrite(RELAY_IGN, HIGH);
  pinMode(RELAY_START, OUTPUT);   digitalWrite(RELAY_START, HIGH);
  pinMode(RELAY_HORN, OUTPUT);    digitalWrite(RELAY_HORN, HIGH);
  pinMode(RELAY_HEAD, OUTPUT);    digitalWrite(RELAY_HEAD, HIGH);

  pinMode(RF_BTN1, INPUT);
  pinMode(RF_BTN2, INPUT);
  pinMode(RF_BTN3, INPUT);
  pinMode(RF_BTN4, INPUT);

  Wire.begin(MPU_SDA, MPU_SCL);
  Serial.begin(115200);
  Serial.println("Anti-Theft ESP32 Starting.");

  // Initialize MPU6050
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x6B);  // PWR_MGMT_1 register
  Wire.write(0);     // Wake up MPU6050
  Wire.endTransmission(true);

  delay(100);
  Serial.println("MPU6050 Initialized.");
}

void loop() {
  bool btnArm = digitalRead(RF_BTN1);
  bool btnStart = digitalRead(RF_BTN2);
  bool btnHorn = digitalRead(RF_BTN3);
  bool btnHead = digitalRead(RF_BTN4);

  static bool lastArm = false;
  if (btnArm && !lastArm) {
    armed = !armed;
    Serial.print("System "); Serial.println(armed ? "Armed" : "Disarmed");
    if (armed) {
      digitalWrite(RELAY_IGN, LOW);    // Lock
      digitalWrite(RELAY_START, HIGH);
    } else {
      digitalWrite(RELAY_IGN, HIGH);
      digitalWrite(RELAY_START, HIGH);
    }
    delay(200);
  }
  lastArm = btnArm;

  if (!armed && btnStart) {
    Serial.println("Self-Start Activated");
    digitalWrite(RELAY_START, LOW);
    delay(1000);
    digitalWrite(RELAY_START, HIGH);
    delay(200);
  }

  if (btnHorn) {
    Serial.println("Horn Activated");
    digitalWrite(RELAY_HORN, LOW);
    delay(500);
    digitalWrite(RELAY_HORN, HIGH);
    delay(300);
  }

  static bool lastHead = false;
  if (btnHead && !lastHead) {
    relayHeadlightOn = !relayHeadlightOn;
    Serial.print("Headlight "); Serial.println(relayHeadlightOn ? "ON" : "OFF");
    digitalWrite(RELAY_HEAD, relayHeadlightOn ? LOW : HIGH);
    delay(200);
  }
  lastHead = btnHead;

  if (armed && detectsMotion()) {
    Serial.println("Motion Detected! Alarm Triggered!");
    for (int i = 0; i < 5; i++) {
      digitalWrite(RELAY_HORN, LOW);
      delay(200);
      digitalWrite(RELAY_HORN, HIGH);
      delay(200);
    }
    delay(1000);
  }

  delay(50);
}

// ---------------- MPU6050 Motion Detection ----------------
bool detectsMotion() {
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x3B); // starting register for accelerometer data
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_ADDR, 6, true);

  int16_t accX = Wire.read() << 8 | Wire.read();
  int16_t accY = Wire.read() << 8 | Wire.read();
  int16_t accZ = Wire.read() << 8 | Wire.read();

  // Compute the magnitude of acceleration vector
  float accelMag = sqrt(accX * accX + accY * accY + accZ * accZ) / 16384.0;

  float diff = fabs(accelMag - lastAccel);
  lastAccel = accelMag;

  // Motion threshold (you can tune this)
  if (diff > 0.25 && millis() - lastMotionTime > 3000) {
    lastMotionTime = millis();
    return true;
  }

  return false;
}
