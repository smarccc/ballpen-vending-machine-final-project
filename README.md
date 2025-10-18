#include <LiquidCrystal_I2C.h>
#include <Servo.h>
#include <Wire.h>

// Create servo objects
Servo myServo1;
Servo myServo2;

// LCD setup
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Pin definitions
const int coinPin = 2;
const int buttonPin = 12;
const int redLEDPin = 10;
const int greenLEDPin = 11;
const int buzzerPin = 9; // Buzzer pin

// IR sensor pins
const int irSensor1Pin = 7;
const int irSensor2Pin = 8;

// Variables
volatile int coinCount = 0;
int buttonState = 0;

// Debounce variables
volatile unsigned long lastCoinInterrupt = 0;
const unsigned long debounceDelay = 300;

// LCD state tracking
String lastLine1 = "";
String lastLine2 = "";

void setup() {
  pinMode(coinPin, INPUT_PULLUP);
  pinMode(buttonPin, INPUT_PULLUP);
  pinMode(redLEDPin, OUTPUT);
  pinMode(greenLEDPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT); // Set buzzer as output

  pinMode(irSensor1Pin, INPUT);
  pinMode(irSensor2Pin, INPUT);

  attachInterrupt(digitalPinToInterrupt(coinPin), coinDetected, FALLING);

  myServo1.attach(5);
  myServo2.attach(6);

  myServo1.write(0);
  myServo2.write(90);

  digitalWrite(redLEDPin, HIGH);
  digitalWrite(greenLEDPin, LOW);
  digitalWrite(buzzerPin, LOW); // Buzzer OFF at start

  Serial.begin(9600);
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor((16 - strlen("5 PESOS BALLPEN")) / 2, 0);
  lcd.print("5 PESOS BALLPEN");
}

void loop() {
  buttonState = digitalRead(buttonPin);

  // Read IR sensors
  int ir1State = digitalRead(irSensor1Pin);
  int ir2State = digitalRead(irSensor2Pin);

  // Log IR sensor status
  Serial.print("IR1: ");
  Serial.print(ir1State == LOW ? "DETECTED" : "CLEAR");
  Serial.print(" | IR2: ");
  Serial.println(ir2State == LOW ? "DETECTED" : "CLEAR");

  // If both sensors are clear (no ballpens available)
  if (ir1State == HIGH && ir2State == HIGH) {
    if (coinCount > 0) {
      coinCount = 0;
      lastLine1 = "";
      lastLine2 = "";
    }

    String line1 = "NO BALLPEN";
    String line2 = "AVAILABLE";
    if (line1 != lastLine1 || line2 != lastLine2) {
      lcd.clear();
      lcd.setCursor((16 - line1.length()) / 2, 0);
      lcd.print(line1);
      lcd.setCursor((16 - line2.length()) / 2, 1);
      lcd.print(line2);
      lastLine1 = line1;
      lastLine2 = line2;
    }

    digitalWrite(redLEDPin, HIGH);
    digitalWrite(greenLEDPin, LOW);
    digitalWrite(buzzerPin, HIGH); // Buzzer ON when no ballpens

    delay(100);
    return;
  } else {
    digitalWrite(buzzerPin, LOW); // Buzzer OFF when ballpens are available
  }

  // Normal display and LED logic
  digitalWrite(redLEDPin, coinCount == 0);
  digitalWrite(greenLEDPin, coinCount != 0);

  String line1 = coinCount == 0 ? "PLEASE INSERT" : "PRESS BUTTON";
  String line2 = coinCount == 0 ? "5 PESOS ONLY" : "INSERTED COIN:" + String(coinCount * 5);

  if (line1 != lastLine1 || line2 != lastLine2) {
    lcd.clear();
    lcd.setCursor((16 - line1.length()) / 2, 0);
    lcd.print(line1);
    lcd.setCursor((16 - line2.length()) / 2, 1);
    lcd.print(line2);
    lastLine1 = line1;
    lastLine2 = line2;
  }

  // Dispense logic
  if (buttonState == LOW && coinCount * 5 >= 5) {
    lcd.clear();
    lcd.setCursor((16 - strlen("Dispensing")) / 2, 0);
    lcd.print("Dispensing");
    lcd.setCursor((16 - strlen("Please wait...")) / 2, 1);
    lcd.print("Please wait...");

    delay(1000);
    myServo2.write(0);
    delay(500);
    myServo1.write(50);
    delay(500);
    myServo1.write(0);
    delay(500);
    myServo2.write(90);
    delay(500);

    coinCount--;
    lastLine1 = "";
    lastLine2 = "";
    while (digitalRead(buttonPin) == LOW);
    delay(200);
  }

  delay(100);
}

// Coin detection with debounce and IR check
void coinDetected() {
  unsigned long currentTime = millis();
  if (currentTime - lastCoinInterrupt > debounceDelay) {
    int ir1State = digitalRead(irSensor1Pin);
    int ir2State = digitalRead(irSensor2Pin);

    // Only count coin if at least one IR sensor detects a ballpen
    if (ir1State == LOW || ir2State == LOW) {
      coinCount++;
      lastCoinInterrupt = currentTime;
    }
  }
}
