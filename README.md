# H2-rover
/****************************************************
 *  COMBINED COLOR-FOLLOWING + DHT11 SENSOR CODE
 ****************************************************/


#include <DHT11.h>
#include "Adafruit_TCS34725.h"


// ------------------ DHT11 Sensor ------------------
//CHANGE CODE
DHT11 dht11(2);


// ================== PINOUT (TB6612FNG) ==================


//CHANGE CODE
const uint8_t PWMA = 9;
const uint8_t AIN1 = 8;
const uint8_t AIN2 = 7;


const uint8_t PWMB = 3;
const uint8_t BIN1 = 5;
const uint8_t BIN2 = 4;


const uint8_t STBY = 6;


// ================== USER CONFIGURATION ==================


//CHANGE CODE
const String FOLLOW_COLORS[] = {"BLACK", "BLUE", "YELLOW"};
const int NUM_FOLLOW_COLORS = sizeof(FOLLOW_COLORS) / sizeof(FOLLOW_COLORS[0]);


//CHANGE CODE
const String LEFT_BOUNDARY_COLOR  = "GREEN";
const String RIGHT_BOUNDARY_COLOR = "RED";


const String LEFT_TURN_MARKER  = "WHITE";
const String RIGHT_TURN_MARKER = "PINK";


// ================== SPEED CONTROLS ==================


//CHANGE CODE
const uint8_t FWD_PWM = 30;
uint8_t PIVL_LEFT_PWM  = 40;
uint8_t PIVL_RIGHT_PWM = 40;
uint8_t PIVR_LEFT_PWM  = 40;
uint8_t PIVR_RIGHT_PWM = 40;


// ================== COLOR SENSOR ==================
Adafruit_TCS34725 tcs(
  TCS34725_INTEGRATIONTIME_50MS,
  TCS34725_GAIN_4X
);


// ================== MOTOR HELPERS ==================
inline uint8_t clampPWM(int v){ return (uint8_t)constrain(v,0,255); }


void leftMotorForward(uint8_t pwm){ digitalWrite(AIN1, HIGH); digitalWrite(AIN2, LOW); analogWrite(PWMA, clampPWM(pwm)); }
void leftMotorBackward(uint8_t pwm){ digitalWrite(AIN1, LOW); digitalWrite(AIN2, HIGH); analogWrite(PWMA, clampPWM(pwm)); }
void leftMotorStop(){ digitalWrite(AIN1, LOW); digitalWrite(AIN2, LOW); analogWrite(PWMA, 0); }


void rightMotorForward(uint8_t pwm){ digitalWrite(BIN1, HIGH); digitalWrite(BIN2, LOW); analogWrite(PWMB, clampPWM(pwm)); }
void rightMotorBackward(uint8_t pwm){ digitalWrite(BIN1, LOW); digitalWrite(BIN2, HIGH); analogWrite(PWMB, clampPWM(pwm)); }
void rightMotorStop(){ digitalWrite(BIN1, LOW); digitalWrite(BIN2, LOW); analogWrite(PWMB, 0); }


void motorsBrake(){
  leftMotorStop();
  rightMotorStop();
}


void motorsForward(uint8_t l, uint8_t r){
  leftMotorBackward(l);
  rightMotorBackward(r);
}
void pivotLeft(uint8_t l, uint8_t r){
  leftMotorBackward(l);
  rightMotorForward(r);
}
void pivotRight(uint8_t l, uint8_t r){
  leftMotorForward(l);
  rightMotorBackward(r);
}


// ================== COLOR DETECTION ==================
String detectColor(float r, float g, float b) {


  //CHANGE CODE
  if ((r >= 115 && r <= 130) && (g >= 85 && g <= 95) && (b >= 55 && b <= 70)) return "BLACK";
  if ((r >= 90 && r <= 100)  && (g >= 80  && g <= 100) && (b >= 60 && b <= 70)) return "WHITE";
  if ((r >= 125 && r <= 145) && (g >= 60  && g <= 75)  && (b >= 50 && b <= 65)) return "PINK";
  if ((r >= 140 && r <= 155) && (g >= 65  && g <= 85)  && (b >= 40 && b <= 60)) return "RED";
  if ((r >= 70  && r <= 100) && (g >= 100 && g <= 140) && (b >= 35 && b <= 65)) return "GREEN";
  if ((r >= 70  && r <= 110) && (g >= 70  && g <= 110) && (b >= 80 && b <= 120)) return "BLUE";
  if ((r >= 110 && r <= 140) && (g >= 85  && g <= 115) && (b >= 20 && b <= 50)) return "YELLOW";
  if ((r >= 120 && r <= 150) && (g >= 8   && g <= 40 ) && (b >= 70 && b <= 110)) return "PURPLE";
  return "UNKNOWN";
}


inline String readColorOnce(float &r, float &g, float &b) {
  tcs.setInterrupt(false); delay(60);
  tcs.getRGB(&r, &g, &b);
  tcs.setInterrupt(true);
  return detectColor(r, g, b);
}


// ================== SETUP ==================
void setup() {
  Serial.begin(9600);


  pinMode(PWMA, OUTPUT); pinMode(AIN1, OUTPUT); pinMode(AIN2, OUTPUT);
  pinMode(PWMB, OUTPUT); pinMode(BIN1, OUTPUT); pinMode(BIN2, OUTPUT);
  pinMode(STBY, OUTPUT);
  digitalWrite(STBY, HIGH);
  motorsBrake();


  if (!tcs.begin()) {
    Serial.println("No TCS34725 found!");
    while(1);
  }


  Serial.println("System Ready: Color Sensor + DHT11");
}


// ================== MAIN LOOP ==================
void loop() {


  /************** READ DHT11 SENSOR **************/
  int temp = 0, hum = 0;
  int result = dht11.readTemperatureHumidity(temp, hum);


  if (result == 0) {
    Serial.print("TEMP: "); Serial.print(temp);
    Serial.print("°C  HUM: "); Serial.print(hum);
    Serial.println("%");
  } else {
    Serial.println(DHT11::getErrorString(result));
  }


  /************** READ COLOR SENSOR **************/
  float r, g, b;
  String color = readColorOnce(r, g, b);


  Serial.print("RGB: ");
  Serial.print(int(r)); Serial.print(",");
  Serial.print(int(g)); Serial.print(",");
  Serial.print(int(b));
  Serial.print(" -> ");
  Serial.println(color);


  if (color == LEFT_TURN_MARKER) {
    motorsBrake(); delay(150);
    pivotLeft(PIVL_LEFT_PWM, PIVL_RIGHT_PWM);
    delay(400);
    motorsBrake(); delay(100);
    return;
  }


  if (color == RIGHT_TURN_MARKER) {
    motorsBrake(); delay(150);
    pivotRight(PIVR_LEFT_PWM, PIVR_RIGHT_PWM);
    delay(400);
    motorsBrake(); delay(100);
    return;
  }


  bool isFollowColor = false;
  for (int i = 0; i < NUM_FOLLOW_COLORS; i++)
    if (color == FOLLOW_COLORS[i]) isFollowColor = true;


  if (isFollowColor) {
    motorsForward(FWD_PWM, FWD_PWM);
    delay(50);
    return;
  }


  if (color == LEFT_BOUNDARY_COLOR) {
    motorsBrake(); delay(150);
    pivotRight(PIVR_LEFT_PWM, PIVR_RIGHT_PWM);
    delay(400);
    motorsBrake(); delay(100);
    return;
  }


  if (color == RIGHT_BOUNDARY_COLOR) {
    motorsBrake(); delay(150);
    pivotLeft(PIVL_LEFT_PWM, PIVL_RIGHT_PWM);
    delay(400);
    motorsBrake(); delay(100);
    return;
  }


  // Recovery sequence
  motorsForward(FWD_PWM, FWD_PWM); delay(250);
  motorsBrake(); delay(100);


  float rN, gN, bN;
  String colorN = readColorOnce(rN, gN, bN);
  for (int i = 0; i < NUM_FOLLOW_COLORS; i++)
    if (colorN == FOLLOW_COLORS[i]) {
      motorsForward(FWD_PWM, FWD_PWM);
      return;
    }


  pivotLeft(PIVL_LEFT_PWM, PIVL_RIGHT_PWM); delay(300);
  motorsBrake(); delay(100);


  float rL, gL, bL;
  String colorL = readColorOnce(rL, gL, bL);
  for (int i = 0; i < NUM_FOLLOW_COLORS; i++)
    if (colorL == FOLLOW_COLORS[i]) {
      motorsForward(FWD_PWM, FWD_PWM);
      return;
    }


  pivotRight(PIVR_LEFT_PWM, PIVR_RIGHT_PWM); delay(600);
  motorsBrake(); delay(100);


  float rR, gR, bR;
  String colorR = readColorOnce(rR, gR, bR);
  for (int i = 0; i < NUM_FOLLOW_COLORS; i++)
    if (colorR == FOLLOW_COLORS[i]) {
      motorsForward(FWD_PWM, FWD_PWM);
      return;
    }


  motorsBrake();
  delay(300);
}
