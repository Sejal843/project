# project
#include <Wire.h>
#define BLYNK_TEMPLATE_ID "TMPL3l_8-ATCp"
#define BLYNK_TEMPLATE_NAME "fall detector"
#define BLYNK_AUTH_TOKEN "utBJrIVv5-AJuFJ1cOWROf8IFWvni4Z1"
#define BLYNK_PRINT Serial
#include <Blynk.h>
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>

// Your Blynk authentication and WiFi credentials
char auth[] = BLYNK_AUTH_TOKEN;
char ssid[] = "Ankitha";
char pass[] = "ankitharao";

// MPU6050 settings
const uint8_t MPU_addr = 0x68;
int16_t AcX, AcY, AcZ;
int minVal = 265;
int maxVal = 402;

// Relay pin (connected to solenoid valve)
#define RELAY_PIN D1

// Angle thresholds for fall detection
double fallThresholdX = 60.0;
double fallThresholdY = 60.0;
double fallThresholdZ = 60.0;

double x, y, z; // Variables to store angles

void setup() {
  Wire.begin();
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x6B);
  Wire.write(0);
  Wire.endTransmission(true);
  
  Serial.begin(115200);
  Blynk.begin(auth, ssid, pass);

  // Set up relay pin
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay off
}

void loop() {
  Blynk.run();

  // Read MPU6050 data
  Wire.beginTransmission(MPU_addr);
  Wire.write(0x3B);
  Wire.endTransmission(false);
  Wire.requestFrom((uint8_t)MPU_addr, (size_t)14, true);
  AcX = Wire.read() << 8 | Wire.read();
  AcY = Wire.read() << 8 | Wire.read();
  AcZ = Wire.read() << 8 | Wire.read();

  // Map accelerometer values to angles
  int xAng = map(AcX, minVal, maxVal, -90, 90);
  int yAng = map(AcY, minVal, maxVal, -90, 90);
  int zAng = map(AcZ, minVal, maxVal, -90, 90);

  // Calculate angles in degrees
  x = RAD_TO_DEG * (atan2(-yAng, -zAng) + PI);
  y = RAD_TO_DEG * (atan2(-xAng, -zAng) + PI);
  z = RAD_TO_DEG * (atan2(-yAng, -xAng) + PI);

  // Display angles on Serial Monitor
  Serial.print("AngleX= ");
  Serial.println(x);
  Serial.print("AngleY= ");
  Serial.println(y);
  Serial.print("AngleZ= ");
  Serial.println(z);
  Serial.println("-----------------------------------------");

  // Send angle data to Blynk app
  Blynk.virtualWrite(V2, x);
  Blynk.virtualWrite(V3, y);
  Blynk.virtualWrite(V4, z);

  // Check if fall detected
  if (x > fallThresholdX || y > fallThresholdY || z > fallThresholdZ) {
    Serial.println("Fall detected! Activating airbag...");

    // Activate relay to open solenoid valve (airbag)
    digitalWrite(RELAY_PIN, HIGH);
    delay(1000); // Keep the solenoid valve open for 1 second
    digitalWrite(RELAY_PIN, LOW);

    // Send fall notification to Blynk app
    Blynk.logEvent("fall_detected", "Airbag deployed due to fall");
    
    // Log fall event to Blynk
    Blynk.virtualWrite(V5, "Fall Detected!");

    // Wait for a while to prevent multiple triggers
    delay(5000);
  }

  // Short delay between readings
  delay(1000);
}
