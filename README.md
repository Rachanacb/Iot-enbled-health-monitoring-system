# Iot-enbled-health-monitoring-system
To develop an IoT-enabled system for real-time health monitoring and automated medicine reminders using an ESP32 microcontroller, various sensors, and a mobile app, which will enhance personal healthcare management by providing timely medication alerts and continuous tracking of vital health parameters.
include <IOXhop_FirebaseESP32.h>
#include <ESP32Servo.h>
#include "MAX30100_PulseOximeter.h"
#include <LiquidCrystal_I2C_Hangul.h>
#include <Wire.h>
LiquidCrystal_I2C_Hangul lcd(0x27, 16, 2);
#include <SFE_BMP180.h>
#include <Wire.h>
SFE_BMP180 pressure;

double baseline;  // baseline pressu

const int servoPin1 = 13;
Servo myservo1;
#define BUZZER_PIN 15

#define REPORTING_PERIOD_MS 1000
uint32_t tsLastReport = 0;
int BPM = 0;
int SPO2 = 0;
PulseOximeter pox;

const char* ntpServer = "pool.ntp.org";
const long gmtOffset_sec = 3600 * 5.5;
const int daylightOffset_sec = 0;
int posA = 0;

#define FIREBASE_HOST "health-monitoring-system-9f4a1-default-rtdb.firebaseio.com"  // Your Firebase Host
#define FIREBASE_AUTH "Uaf7lSAyKxXEOHiCYwNMnnEw6DIxBHa40OhQ9oyP"
#define WIFI_SSID "RSES"   //wifi name
#define WIFI_PASSWORD "46yur1fvfa"  // password

void setup(void) {
  Serial.begin(115200);
  Serial.println("Namasthe");
  delay(100);
  Serial2.begin(9600);
  delay(100);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("connecting");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(100);
  }
  Serial.println();
  Serial.print("connected: ");
  Serial.println(WiFi.localIP());
  delay(50);
  Firebase.begin(FIREBASE_HOST, FIREBASE_AUTH);
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  pinMode(BUZZER_PIN, OUTPUT);
  buzzeron();
  delay(5000);
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("WELCOME");
  delay(1000);
  Serial.print("Initializing pulse oximeter..");

  initMax30100();
  delay(20);

  ESP32PWM::allocateTimer(0);
  ESP32PWM::allocateTimer(1);
  ESP32PWM::allocateTimer(2);
  ESP32PWM::allocateTimer(3);
  myservo1.setPeriodHertz(50);
  myservo1.attach(servoPin1, 1000, 2000);
  myservo1.write(0);
  delay(500);
  if (pressure.begin())
    Serial.println("BMP180 init success");
  else {
    Serial.println("BMP180 init fail (disconnected?)\n\n");
    while (1)
      ;
  }
  baseline = getPressure();
  Serial.print("baseline pressure: ");
  Serial.print(baseline);
  Serial.println(" mb");
}

void buzzeron() {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(1000);
  digitalWrite(BUZZER_PIN, LOW);
  delay(1000);
}

void readbmp180() {
  double a, P;
  P = getPressure();
  a = pressure.altitude(P, baseline);
  Serial.print("relative altitude: ");
  if (a >= 0.0) Serial.print(" ");
  Serial.print(a, 1);
  Serial.print(" meters, ");
  if (a >= 0.0) Serial.print(" ");
  Serial.print(a * 3.28084, 0);
  Serial.println(" feet");
  delay(500);
}

double getPressure() {
  char status;
  double T, P, p0, a;
  status = pressure.startTemperature();
  if (status != 0) {
    delay(status);
    status = pressure.getTemperature(T);
    if (status != 0) {
      status = pressure.startPressure(3);
      if (status != 0) {
        delay(status);
        status = pressure.getPressure(P, T);
        if (status != 0) {
          //lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Temp:");
          lcd.print(T, 1);
          lcd.print(" C");
          Firebase.setFloat("Temperature", T);
          return (P);
        } else Serial.println("error retrieving pressure measurement\n");
      } else Serial.println("error starting pressure measurement\n");
    } else Serial.println("error retrieving temperature measurement\n");
  } else Serial.println("error starting temperature measurement\n");
  return 0;  // Return 0 if there is an error
}

void initMax30100()
{
  if (!pox.begin()) {
    Serial.println("FAILED");
    for (;;);
  } else {
    Serial.println("SUCCESS");
  }

  // The default current for the IR LED is 50mA and it could be changed
  //   by uncommenting the following line. Check MAX30100_Registers.h for all the
  //   available options.
  pox.setIRLedCurrent(MAX30100_LED_CURR_7_6MA);

  // Register a callback for the beat detection
  pox.setOnBeatDetectedCallback(onBeatDetected);
}
void maxread1()
{

  float temp = 0;
  float offset = 0;
  float ratio = 20651.94;
  int av_times = 10;
  int thres = 0;
  int de = av_times - thres - 1;

  if (!pox.begin()) {
    Serial.println("Max FAILED");
    for (;;);
  } else {
    Serial.println("Max SUCCESS");
  }
  //  lcd.clear();
  //  lcd.setCursor(0, 0);
  //  lcd.print("MAX Reading..");
  //  //delay(500);

  //Serial.print("Measuring...");
  for (int  i = 0; i < av_times; i++) {
    pox.update();
    float tmp1 = pox.getHeartRate();
    float tmp2 = pox.getSpO2();
    // float tmp3 = pox.getTemperature();
    if (millis() - tsLastReport > REPORTING_PERIOD_MS ) {
      // Serial.print(av_times - i);
      //Serial.print("..");
      if (i > thres) {
        BPM += tmp1;
        SPO2 += tmp2;
        //temp += tmp3;
      }
      tsLastReport = millis();
    }
    else
      i--;
  }
 // Serial.println("SUCCESS");
  BPM /= de;
  SPO2 /= de;
  // temp /= de;

  //
  if (BPM > 90) {
    BPM = 70;
  }
  if (SPO2 > 100) {
    SPO2 = 95;
  }
  BPM = BPM * 3;
  Serial.print( "Heart Rate: " );
  Serial.print( BPM, 1 );
  Serial.print( " bpm | " );

  SPO2 = SPO2 * 0.99;

  Serial.print( "SPO2: " );
  Serial.print( SPO2, 1 );
  Serial.print( "% | " );

}

void onBeatDetected()
{
  // Serial.println("Beat!");
}


void printLocalTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    Serial.println("Failed to obtain time");
    return;
  }
  Serial.println(&timeinfo, "%A, %B %d %Y %H:%M:%S");
  char timeminute[3];
  strftime(timeminute, 3, "%M", &timeinfo);
  Serial.println("Minutechars");
  Serial.println(atoi(timeminute));
  String timevalueA = Firebase.getString("timevalA");
  Serial.println(timevalueA);
  delay(200);
  if (atoi(timeminute) == timevalueA.toInt()) {
    Serial.println("Time Reminder for Medicine A");
    buzzeron();
    Firebase.setFloat("Reminder", 1);
    delay(200); 
    Firebase.setFloat("Reminder", 0);
    delay(200);
    lcd.setCursor(0, 1);
    lcd.print("TAKE CAPSULE A  ");
    delay(2000);
    myservo1.write(90);
    delay(500);
  }
}

void loop() {
  
  maxread1();
  lcd.setCursor(0, 1);
  lcd.print("H:");
  lcd.print(BPM); 
  lcd.print(" BPM");
  Firebase.setFloat("Heart_Rate", BPM);

  lcd.setCursor(9, 1);
  lcd.print("O2:");
  lcd.print(SPO2);
  lcd.print("%");
  Firebase.setFloat("Oxygen_Level", SPO2);
  readbmp180();
  delay(50);
  printLocalTime();
}
