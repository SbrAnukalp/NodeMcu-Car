#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <Servo.h>
#include <FS.h>

const char* apSSID = "NodeMCU_Car";
const char* apPassword = "password";

ESP8266WebServer server(80);

Servo steeringServo;

// Pin definitions
const int motorPin1 = D1; // Motor driver input 1
const int motorPin2 = D2; // Motor driver input 2
const int servoPin = D3;  // Servo pin for left-right movement
const int ledPin = D5;    // LED pin

bool isTurningLeft = false;
bool isTurningRight = false;

void handleRoot() {
    File htmlFile = SPIFFS.open("/car_control.html", "r");
    if (htmlFile) {
        server.streamFile(htmlFile, "text/html");
        htmlFile.close();
    } else {
        server.send(404, "text/plain", "File not found");
    }
}

void handleForward() {
    digitalWrite(motorPin1, HIGH);
    digitalWrite(motorPin2, LOW);
    server.send(200, "text/plain", "Moving forward");
}

void handleBackward() {
    digitalWrite(motorPin1, LOW);
    digitalWrite(motorPin2, HIGH);
    server.send(200, "text/plain", "Moving backward");
}

void handleBrake() {
    digitalWrite(motorPin1, LOW);
    digitalWrite(motorPin2, LOW);
    server.send(200, "text/plain", "Braking");
}

void handleLedOn() {
    digitalWrite(ledPin, HIGH);
    server.send(200, "text/plain", "LED turned on");
}

void handleLedOff() {
    digitalWrite(ledPin, LOW);
    server.send(200, "text/plain", "LED turned off");
}

void handleLeft() {
    isTurningLeft = true;
    isTurningRight = false;
    steeringServo.write(0); // Turn left
    server.send(200, "text/plain", "Turning left");
}

void handleRight() {
    isTurningLeft = false;
    isTurningRight = true;
    steeringServo.write(180); // Turn right
    server.send(200, "text/plain", "Turning right");
}

void handleStopTurning() {
    isTurningLeft = false;
    isTurningRight = false;
    // Set servo to neutral position (90 degrees)
    steeringServo.write(90);
    server.send(200, "text/plain", "Stopped turning");
}

void handleJoystick() {
    int posX = server.arg("x").toInt(); // Get X position of joystick
    int posY = server.arg("y").toInt(); // Get Y position of joystick

    // Map joystick position to servo angle (0 to 180 degrees)
    int angle = map(posX, 0, 100, 0, 180); // Adjust map range as needed

    steeringServo.write(angle);
    server.send(200, "text/plain", "Servo position set");
}

void setup() {
    Serial.begin(115200);

    pinMode(motorPin1, OUTPUT);
    pinMode(motorPin2, OUTPUT);
    pinMode(ledPin, OUTPUT);

    steeringServo.attach(servoPin);

    WiFi.softAP(apSSID, apPassword);

    IPAddress apIP = WiFi.softAPIP();
    Serial.print("AP IP address: ");
    Serial.println(apIP);

    if (SPIFFS.begin()) {
        Serial.println("File system mounted");
        server.on("/", handleRoot);
    } else {
        Serial.println("Failed to mount file system");
    }

    server.on("/", handleRoot);
    server.on("/forward", handleForward);
    server.on("/backward", handleBackward);
    server.on("/brake", handleBrake);
    server.on("/left", handleLeft);
    server.on("/right", handleRight);
    server.on("/stopturning", handleStopTurning);
    server.on("/joystick", handleJoystick); // New handler for joystick position
    server.on("/led/on", handleLedOn);
    server.on("/led/off", handleLedOff);

    server.begin();
}

void loop() {
    server.handleClient();

    // Check if left or right button released
    if (!isTurningLeft && !isTurningRight) {
        handleStopTurning();
    }
}