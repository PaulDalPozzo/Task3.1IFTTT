#include <WiFiNINA.h>
#include <BH1750.h>
#include <Wire.h>
#include "secrets.h"

char ssid[] = SECRET_SSID;   // your network SSID (name) 
char pass[] = SECRET_PASS;   // your network password
int status = WL_IDLE_STATUS;     // the WiFi radio's status
int keyIndex = 0;            // your network key Index number (needed only for WEP)
WiFiClient  client;

// IFTTT setup
char   HOST_NAME[] = "maker.ifttt.com";
String PATH_ONE   = "/trigger/"; // change your EVENT-NAME and YOUR-KEY
String PATH_TWO   = "/with/key/bdPJr_muRI-qiKcQswflfT"; // change your EVENT-NAME and YOUR-KEY
String queryString = "";
//in sunlight https://maker.ifttt.com/trigger/in_sunlight/with/key/bdPJr_muRI-qiKcQswflfT
//in darkness https://maker.ifttt.com/trigger/in_darkness/with/key/bdPJr_muRI-qiKcQswflfT

// state setup
const int INITIALIZE = 0, IN_DARK = 1, IN_SUNLIGHT = 2;
int _state;
bool in_sunlight = false;
const int SUNLIGHT_LUX = 30;

// timer parameters in milliseconds
const double INTERVAL_TIME = 500; //millis() returns between 0 and 255 (1s), happens between each dot and dash

// timers
double time_in_sun = 0;
int seconds = 0;
int minutes = 0;
int hours = 0;
unsigned long currentMillis = 0;
unsigned long previousMillis = 0;

// light sensor
BH1750 lightMeter;

void setup() {
  Serial.begin(115200);      // Initialize serial 
  while (!Serial) {
    ; // wait for serial port to connect. Needed for Leonardo native USB port only
  }

  // check for the WiFi module:
  if (WiFi.status() == WL_NO_MODULE) {
    Serial.println("Communication with WiFi module failed!");
    // don't continue
    while (true);
  }

  String fv = WiFi.firmwareVersion();
  if (fv < WIFI_FIRMWARE_LATEST_VERSION) {
    Serial.println("Please upgrade the firmware.");
  }

  // attempt to connect to WiFi network:
  while (status != WL_CONNECTED) {
    Serial.print("Attempting to connect to WPA SSID: ");
    Serial.println(ssid);
    // Connect to WPA/WPA2 network:
    status = WiFi.begin(ssid, pass);

    // wait 10 seconds for connection:
    loadingDelay(10);
  }

  // you're connected now, so print out the data:
  Serial.println("You're connected to the network.");
  

  Serial.begin(9600);   // Initialize serial 
  Wire.begin();         // Initialize the I2C bus (BH1750 library doesn't do this automatically)
  lightMeter.begin();   // Initialize light meter 
  next(INITIALIZE);
}

void loop() {
  // timer to run on INTERVAL_TIME - set to 128 (0.5 seconds)
  currentMillis = millis() - previousMillis;
  if (currentMillis < INTERVAL_TIME + previousMillis) return;
  previousMillis = currentMillis;
  readSensor();
  tick();
  
  // Connect or reconnect to WiFi
  if(WiFi.status() != WL_CONNECTED){
    Serial.print("Attempting to connect to SSID: ");
    Serial.println(SECRET_SSID);
    while(WiFi.status() != WL_CONNECTED){
      WiFi.begin(ssid, pass);  // Connect to WPA/WPA2 network. Change this line if using open or WEP network
      Serial.print(".");
      loadingDelay(5);
    } 
    Serial.println("\nConnected.");
  }
}

void loadingDelay(int time){
  for (int i = 0; i < time; i++){
    Serial.print(". ");
    delay(1000);
  }
  Serial.println(". ");
}

void buildQuery(){
  queryString = "?value1=";
  queryString += hours;
  queryString += "&value2=";
  queryString += minutes;
  queryString += "&value3=";
  queryString += seconds;
}

void sendWebhook(){
  Serial.println("Connecting to Server.");
  
  // connect to web server on port 80:
  if (client.connect(HOST_NAME, 80)) {
    // if connected:
    Serial.println("Connected to server");
  }
  else {// if not connected:
    Serial.println("Connection failed");
  }

  String path_name = PATH_ONE;
  if (in_sunlight) path_name += "in_sunlight";
  else path_name += "in_darkness";

  path_name += PATH_TWO;

  if (!in_sunlight) path_name += queryString;

  Serial.println(path_name);
  // make a HTTP request:
  // send HTTP header
  client.println("POST " + path_name + " HTTP/1.1");
  client.println("Host: " + String(HOST_NAME));
  client.println("Connection: close");
  client.println(); // end HTTP header

  // the server's disconnected, stop the client:
  client.stop();
  Serial.println("Disconnected from Server.");
}

// handles every tick for every state
void tick()
{
  switch(_state)
  {
    case INITIALIZE:
      next(IN_DARK);
      return;
    case IN_DARK:
      return;
    case IN_SUNLIGHT:
      updateTimeInSun();
      //printTimeInSun();
      return;
    default:
      next(INITIALIZE);
  }
}

// resets time in the sun variable to 0
void resetTimeInSun()
{
  seconds = 0;
  minutes = 0;
  hours = 0;
}

// updates the time in the sun
// converts milliseconds to seconds, seconds to minutes, minutes to hours
void updateTimeInSun(){
  time_in_sun += INTERVAL_TIME;
  
  if(time_in_sun > 1000)
  {
    time_in_sun -= 1000;
    seconds++;
  }

  if(seconds == 60)
  {
    seconds = 0;
    minutes++;
  }

  if (minutes == 60) 
  {
    minutes = 0;
    hours++;
  }

  if (hours > 1000) resetTimeInSun();
}

// prints the time in the sun to the serial
void printTimeInSun(){
  Serial.print(hours);
  Serial.print(":");
  Serial.print(minutes);
  Serial.print(":");
  Serial.print(seconds);
  Serial.print(":");
  Serial.println(time_in_sun);
}

// gets an updated reading from the light sensor and handles any changes in state caused my the reading
void readSensor()
{
  // read light level from sensor
  float lux = lightMeter.readLightLevel();
  switch(_state)
  {
    case INITIALIZE:
      return;
    case IN_DARK:
      if (lux > SUNLIGHT_LUX) next(IN_SUNLIGHT);
      return;
    case IN_SUNLIGHT:
      if (lux < SUNLIGHT_LUX) next(IN_DARK);
      return;
  }
}

// handles moving to the next state
void next(int newState)
{
  _state = newState;
  switch(_state)
  {
    case INITIALIZE:
      return;
    case IN_DARK:
      Serial.println("IN DARKNESS");
      Serial.print("Was in sunlight for: ");
      printTimeInSun();
      in_sunlight = false;
      // send email
      buildQuery();
      sendWebhook();
      return;
    case IN_SUNLIGHT:
      Serial.println("=== IN SUNLIGHT ===");
      in_sunlight = true;
      // send email
      sendWebhook();
      return;
  }
}

