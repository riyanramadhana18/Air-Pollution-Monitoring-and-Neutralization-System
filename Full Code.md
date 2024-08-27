Baru! Pintasan keyboard … Pintasan keyboard Drive telah diperbarui untuk memberi Anda navigasi huruf pertama
#include <WiFi.h>
#include "secrets.h"
#include "ThingSpeak.h" // always include thingspeak header file after other header files and custom macros
#include <MQUnifiedsensor.h>

#define PIN_MQ135 35 
#define pinKipasMQ 33
#define pinKipasDust 32
#define VOLTAGE_RESOLUTION 5
#define ADC_BIT_RESOLUTION 10
#define RATIO_AIR_CLEAN 9.83 

char ssid[] = "Riyan1818";   // your network SSID (name) 
char pass[] = "12345678";   // your network password
int keyIndex = 0;            // your network key Index number (needed only for WEP)
WiFiClient  client;

unsigned long myChannelNumber = 2473403;
const char * myWriteAPIKey = "O1FTA5H2F212QR93";

const int pinSensorDebu = 34;  

MQUnifiedsensor MQ135("ESP-32", VOLTAGE_RESOLUTION, ADC_BIT_RESOLUTION, PIN_MQ135, "MQ-135");

void setup() {
  Serial.begin(115200);  //Initialize serial
  while (!Serial) {
    ; // wait for serial port to connect. Needed for Leonardo native USB port only
  }
  MQ135.setRegressionMethod(1); // Pilih metode regresi untuk sensor
  MQ135.setA(102.2); // Parameter A dari kurva regresi (dari datasheet)
  MQ135.setB(-2.473); // Parameter B dari kurva regresi (dari datasheet)
  MQ135.init(); // Inisialisasi sensor
  Serial.print("Calibrating please wait.");
  float calcR0 = 0;
  for (int i = 1; i <= 10; i++) {
    MQ135.update(); // Perbarui nilai sensor
    calcR0 += MQ135.calibrate(RATIO_AIR_CLEAN); // Kalibrasi sensor dengan rasio udara bersih
    Serial.print(".");
  }
  MQ135.setR0(calcR0 / 10); // Set nilai kalibrasi
  Serial.println("done!.");
   if (isinf(calcR0)) {
    Serial.println("Warning: Connection issue or invalid R0");
    while (1);
  }
  if (calcR0 == 0) {
    Serial.println("Warning: Invalid R0");
    while (1);
  }
  pinMode(pinKipasMQ, OUTPUT);
  pinMode(pinKipasDust, OUTPUT);
  WiFi.mode(WIFI_STA);   
  ThingSpeak.begin(client);  // Initialize ThingSpeak
}

void loop() {

  // Connect or reconnect to WiFi
  if(WiFi.status() != WL_CONNECTED){
    Serial.print("Attempting to connect to SSID: ");
    Serial.println(SECRET_SSID);
    while(WiFi.status() != WL_CONNECTED){
      WiFi.begin(ssid, pass);  // Connect to WPA/WPA2 network. Change this line if using open or WEP network
      Serial.print(".");
      delay(5000);     
    } 
    Serial.println("\nConnected.");
  }
   MQ135.update(); // Perbarui nilai sensor
  float ppm = MQ135.readSensor(); // Baca nilai CO dalam ppm
  int MQppm = ppm;
  Serial.print("CO PPM : ");
  Serial.println(MQppm);
  
  int valorSensor = analogRead(pinSensorDebu);
  float voltsDebu = valorSensor * (5.0 / 1024.0); // Mengubah nilai menjadi voltase
  float dustDensity = (0.17 * voltsDebu - 0.38) * 10;
  int debu = dustDensity;

  Serial.print("Kepadatan debu: ");
  Serial.print(debu); // Menampilkan kerapatan debu dalam ug/m3
  Serial.println(" ug/m3");

  digitalWrite(pinKipasMQ, LOW);
  if(MQppm > 8){
    digitalWrite(pinKipasMQ, HIGH);
    }else{
    digitalWrite(pinKipasMQ, LOW);
    }

  digitalWrite(pinKipasDust, LOW);
    if(debu > 9){
    digitalWrite(pinKipasDust, HIGH);
    }else{
    digitalWrite(pinKipasDust, LOW);  
    }

  // set the fields with the values
  ThingSpeak.setField(1, MQppm);
  ThingSpeak.setField(2, debu);
 
  
  // write to the ThingSpeak channel
  int x = ThingSpeak.writeFields(myChannelNumber, myWriteAPIKey);
  if(x == 200){
    Serial.println("Channel update successful.");
  }
  else{
    Serial.println("Problem updating channel. HTTP error code " + String(x));
  }
  
  
  delay(20000); // Wait 20 seconds to update the channel again
}
