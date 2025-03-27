 //RFID based door lock systme code is here......
 #include <WiFi.h>
#include <HTTPClient.h>
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>               // Include the Wire library for I2C
#include <LiquidCrystal_I2C.h>  // Include the LiquidCrystal_I2C library
#include <ESP32Servo.h>         // ESP32-specific Servo library  

// WiFi Configuration
const char* ssid = "SMARTAXIOM";
const char* password = "Amit1305";

// Google Sheets Configuration
String API_ID = "AKfycbx_CdGcPLL_yK7JOcREl8a2SMrHg_eLwzIVy65zerpRapniwNnhCbkE-ASVvjrTjC3Wvg";

// RFID Configuration
#define RST_PIN 22
#define SS_PIN 21
MFRC522 mfrc522(SS_PIN, RST_PIN);
#define ir_sensor 25 // Button to determine IN/OUT
// Button Configuration
#define BUTTON_PIN 15 // Button to determine IN/OUT

// LCD Configuration
#define I2C_SDA 27    // I2C SDA pin
#define I2C_SCL 26    // I2C SCL pin
LiquidCrystal_I2C lcd(0x27, 16, 2); // Initialize the LCD (address, columns, rows)

// LED and Buzzer Configuration
#define RED     2    // Red LED pin
#define GREEN   4    // Green LED pin
#define BUZZER 32    // Buzzer pin
#define RELAY_PIN 13  // Using GPIO13 (D13)
// Servo Configuration
#define SERVO_PIN 33 // Servo motor pin
Servo myServo;       // Create a servo object

void setup() {
  Serial.begin(115200);

  // Initialize WiFi
  connectToWiFi();

  // Initialize RFID
  SPI.begin();
  mfrc522.PCD_Init();

  // Initialize Button
  pinMode(BUTTON_PIN, INPUT); // Button with external pull-up resistor
  pinMode(ir_sensor, INPUT); // Initialize IR sensor as input
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);  // Start with relay OFF
  // Initialize LCD
  Wire.begin(I2C_SDA, I2C_SCL); // Initialize I2C communication
  lcd.init();                   // Initialize the LCD
  lcd.backlight();               // Turn on the backlight
  lcd.setCursor(0, 0);           // Set cursor to the first column, first row
  lcd.print("System Ready!");    // Display a welcome message

  // Initialize LEDs and Buzzer
  pinMode(RED, OUTPUT);
  pinMode(GREEN, OUTPUT);
  pinMode(BUZZER, OUTPUT);

  // Initialize Servo
  myServo.attach(SERVO_PIN); // Attach the servo to the specified pin
  myServo.write(110);        // Set initial position (door closed)

  // Turn off LEDs and Buzzer initially
  digitalWrite(RED, LOW);
  digitalWrite(GREEN, LOW);
  digitalWrite(BUZZER, LOW);

  Serial.println("System Ready!");
}

void loop() {
  rfid_read(); // Handle RFID card scanning
}

// WiFi Connection Handler
void connectToWiFi() {
  Serial.println("\n=== Connecting to WiFi ===");
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\nWiFi Connected!");
  Serial.printf("IP Address: %s\n", WiFi.localIP().toString().c_str());
}

// Data Transmission Core
void sendData(String uid, String name, String mode) {
  HTTPClient http;

  // Construct URL with all parameters
  String url = "https://script.google.com/macros/s/" + API_ID + "/exec";
  url += "?uid=" + uid;
  url += "&name=" + name;
  url += "&mode=" + mode;

  Serial.println("\n=== Sending Data ===");
  Serial.println("URL: " + url);

  http.begin(url); // Initialize HTTP connection
  http.setFollowRedirects(HTTPC_STRICT_FOLLOW_REDIRECTS); // Follow redirects

  int httpCode = http.GET(); // Send GET request

  if (httpCode > 0) {
    Serial.printf("HTTP Status Code: %d\n", httpCode);
    String payload = http.getString();
    Serial.println("Response from server:");
    Serial.println(payload);
  } else {
    Serial.printf("HTTP Request failed. Error: %s\n", http.errorToString(httpCode).c_str());
  }

  http.end(); // Close connection
}

// RFID Reading Function
void rfid_read() {
  // Look for new cards
  if (!mfrc522.PICC_IsNewCardPresent()) return;

  if (!mfrc522.PICC_ReadCardSerial()) return;

  // Read UID and format without spaces
  String uid = "";
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    uid += String(mfrc522.uid.uidByte[i] < 0x10 ? "0" : "");
    uid += String(mfrc522.uid.uidByte[i], HEX);
  }
  uid.toUpperCase();

  // Determine mode based on button state
  String mode = digitalRead(BUTTON_PIN) == HIGH ? "IN" : "OUT";

  // Display UID and Mode on Serial Monitor
  Serial.print("UID tag: ");
  Serial.println(uid);

  Serial.print("Mode: ");
  Serial.println(mode);

  // Display User Info on LCD
  lcd.clear(); // Clear the LCD
  lcd.setCursor(0, 0); // Set cursor to the first column, first row
  lcd.print("ID: " + uid); // Display the UID
  lcd.setCursor(0, 1); // Set cursor to the first column, second row
  lcd.print("Mode: " + mode); // Display the mode (IN/OUT)

  // Control LEDs, Buzzer, and Servo
  if (uid == "03A2AE15") { // Known user: Aditya
    sendData(uid, "Aditya", mode);
    lcd.setCursor(0, 1); // Set cursor to the first column, second row
    lcd.print("User: Aditya"); // Display the user's name

    // Activate Servo (open and close door)
    motor1();
    delay(1000);
    if (mode == "IN") {

      digitalWrite(GREEN, HIGH); // Turn on Green LED
      tone(BUZZER, 1000, 500);  // Beep the buzzer for 500ms
      delay(1000);              // Keep Green LED on for 1 second
      digitalWrite(GREEN, LOW); // Turn off Green LED

    } else {
      digitalWrite(RED, HIGH);  // Turn on Red LED
      tone(BUZZER, 2000, 500);  // Beep the buzzer for 500ms
      delay(1000);              // Keep Red LED on for 1 second
      digitalWrite(RED, LOW);   // Turn off Red LED
    }
  } else if (uid == "03162511") { // Known user: Aman
    sendData("03A2AE16", "Aman", mode);
    lcd.setCursor(0, 1); // Set cursor to the first column, second row
    lcd.print("User: Aman"); // Display the user's name

    // Activate Servo (open and close door)
    motor1();
    delay(1000);
    if (mode == "IN") {

      digitalWrite(GREEN, HIGH); // Turn on Green LED
      tone(BUZZER, 1000, 500);  // Beep the buzzer for 500ms
      delay(1000);              // Keep Green LED on for 1 second
      digitalWrite(GREEN, LOW); // Turn off Green LED

    } else {
      digitalWrite(RED, HIGH);  // Turn on Red LED
      tone(BUZZER, 2000, 500);  // Beep the buzzer for 500ms
      delay(1000);              // Keep Red LED on for 1 second
      digitalWrite(RED, LOW);   // Turn off Red LED
    }
  } else { // Unknown user
    Serial.println("Unknown Card!");
    lcd.setCursor(0, 1); // Set cursor to the first column, second row
    lcd.print("Unknown User!"); // Display unknown user message

    digitalWrite(RED, HIGH);  // Turn on Red LED
    tone(BUZZER, 3000, 1000); // Beep the buzzer for 1 second
    delay(1000);              // Keep Red LED on for 1 second
    digitalWrite(RED, LOW);   // Turn off Red LED
  }

  // Reset RFID reader for next scan
  mfrc522.PICC_HaltA();
  mfrc522.PCD_StopCrypto1(); // Stop encryption (important for reset)
}

// Servo Control Function
void motor1()
{
  // Ensure the servo is attached
  myServo.attach(SERVO_PIN);
  digitalWrite(RELAY_PIN, HIGH); 
  // Open the door (move to 0°)
  for (int pos = 120; pos >= 0; pos -= 2) {
    myServo.write(pos);
    delay(15);
  }

  // Keep door open as long as object is detected
  while (digitalRead(ir_sensor) == LOW) { // Assuming HIGH means object detected
    delay(100); // Small delay to prevent busy-waiting
  }

  digitalWrite(RELAY_PIN, LOW);
  // Close the door (return to 120°)
  for (int pos = 0; pos <= 120; pos += 2) {
    myServo.write(pos);
    delay(15);
  }

  // Detach the servo to save power
  myServo.detach();
}
