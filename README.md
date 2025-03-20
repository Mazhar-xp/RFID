#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_ST7735.h>
#include <MFRC522.h>
#include <Servo.h>  // Include Servo library

// TFT SPI Pins
#define TFT_CS   7   // Chip Select for TFT
#define TFT_DC   6   // Data/Command for TFT
#define TFT_RST  5   // Reset for TFT

// RFID SPI Pins
#define RFID_CS  10  // Chip Select for RFID
#define RFID_RST 9   // Reset for RFID

// Servo Motor Pin
#define SERVO_PIN 3  // Define servo pin (change if needed)

// TFT, RFID, and Servo Objects
Adafruit_ST7735 tft = Adafruit_ST7735(TFT_CS, TFT_DC, TFT_RST);
MFRC522 mfrc522(RFID_CS, RFID_RST);
Servo doorServo;  // Create servo object

// Allowed card UIDs
const byte allowedUIDs[][4] = {
    {0x73, 0xDC, 0xDA, 0xA5},
    {0x33, 0x68, 0xAA, 0xA9},
    {0xE3, 0xE8, 0x1C, 0x2A}
};
const int numAllowedUIDs = sizeof(allowedUIDs) / sizeof(allowedUIDs[0]);

void setup() {
    Serial.begin(9600);
    SPI.begin();

    // Initialize TFT
    pinMode(TFT_CS, OUTPUT);
    digitalWrite(TFT_CS, HIGH);  // Keep disabled by default
    tft.initR(INITR_GREENTAB);
    tft.setRotation(1);
    Serial.println("TFT Initialized.");

    // Initialize RFID
    pinMode(RFID_CS, OUTPUT);
    digitalWrite(RFID_CS, HIGH); // Keep disabled by default
    mfrc522.PCD_Init();
    Serial.println("RFID Initialized.");

    // Initialize Servo
    doorServo.attach(SERVO_PIN);
    doorServo.write(90);  // Start at 0° position

    // Show startup message on TFT
    showStartupScreen();
}

void loop() {
    // Enable RFID, Disable TFT
    digitalWrite(TFT_CS, HIGH);
    digitalWrite(RFID_CS, LOW);

    if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
        Serial.print("Card UID: ");
        bool accessGranted = false;
        for (byte i = 0; i < mfrc522.uid.size; i++) {
            Serial.print(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " ");
            Serial.print(mfrc522.uid.uidByte[i], HEX);
        }
        Serial.println();

        // Check if UID is in the allowed list
        for (int i = 0; i < numAllowedUIDs; i++) {
            if (memcmp(mfrc522.uid.uidByte, allowedUIDs[i], 4) == 0) {
                accessGranted = true;
                break;
            }
        }

        // Enable TFT, Disable RFID
        digitalWrite(RFID_CS, HIGH);
        digitalWrite(TFT_CS, LOW);

        if (accessGranted) {
            showAccessGranted();
            activateServo();  // Call function to move the servo
        } else {
            showAccessDenied();
        }

        // Halt RFID
        mfrc522.PICC_HaltA();
        delay(2000);  // Wait before scanning again
        showStartupScreen();
    }
}

void showStartupScreen() {
    tft.fillScreen(ST7735_BLACK);
    tft.setTextColor(ST7735_WHITE, ST7735_BLACK);
    
    tft.setCursor(20, 20);
    tft.setTextSize(2);
    tft.print("Place the      Card");
    tft.setCursor(40, 70);
    tft.setTextSize(4);
    tft.print("^__^");  // Smiley face
}

void showAccessGranted() {
    tft.fillScreen(ST7735_BLACK);
    tft.setTextColor(ST7735_GREEN);
    tft.setTextSize(2);
    tft.setCursor(20, 20);
    tft.print("  Access      Granted!");
    tft.setCursor(40, 70);
    tft.setTextSize(4);
    tft.print("^O^");  // Big smile
}

void showAccessDenied() {
    tft.fillScreen(ST7735_BLACK);
    tft.setTextColor(ST7735_RED);
    tft.setTextSize(2);
    tft.setCursor(20, 20);
    tft.print("  Access      Denied!");
    tft.setCursor(40, 70);
    tft.setTextSize(4);
    tft.print(">w<");  // Sad face
}

void activateServo() {
    Serial.println("🔓 Unlocking door...");
    doorServo.write(180);  // Rotate to unlock
    delay(200);  // Run for 0.5 seconds
    doorServo.write(90);  // Stop
    delay(5000);  // Wait for 5 seconds

    Serial.println("🔒 Locking door...");
    doorServo.write(0);  // Rotate back to lock
    delay(200);  // Run for 0.5 seconds
    doorServo.write(90);  // Stop
}

