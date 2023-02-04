# pvpc
Visualizador coste en tiempo real del coste del Kw/h de la electricidad PVPC

Este proyecto nos va a permitir saber el coste de la electricidad PVPC

Foto del prototipo:
![image](https://user-images.githubusercontent.com/48222471/216771040-a1290710-e25a-4d1b-8bd3-cba4f3a47e85.png)

Esquema electrico:
![image](https://user-images.githubusercontent.com/48222471/216771065-38075b54-1081-4251-bbe6-352c2559182e.png)

Los pulsadores nos van a permitir visualizar el precio en cada una de las horas del dia.

Este montaje necesita conexion a internet, pan comido para el wifi del ESP32, y una vez conectado puede obtener los datos referentes al día, hora, mes y año y los  datos referentes al coste de la electricidad PVPC. 
Dichos datos se extraen directamente de REE (Red Electrica Española) en https://api.esios.ree.es/archives/70/download_json?locale=es.
Esto nos permite descargar un archivo JSON del que podemos obtener el coste de la electricidad PVPC.
Los datos se actualizan para el día siguiente a partir de las 20:15. Es decir si nos descargamos el JSON antes de las 20:15 obtenemos los datos del dia en curso y a partir de esa hora, los de mañana.

Para deserializar el archivo JSON he utilizado la web https://arduinojson.org/v6/assistant/#/ que me facilita mucho la parte de programación sobre el JSON.

El soft es el siguiente y solo necesitas configurar el SSID y la password:


#include <Arduino.h>
#include <WiFi.h>
#include <WiFiMulti.h>
WiFiMulti wifiMulti;
#include <NTPClient.h>
#include <Time.h>
#include <TimeLib.h>
#include <Timezone.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

//Libraries for LoRa
#include <SPI.h>

//Libraries for OLED Display
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

//OLED pins
#define OLED_SDA 21
#define OLED_SCL 22
#define SCREEN_WIDTH 128 // OLED display width, in pixels
#define SCREEN_HEIGHT 64 // OLED display height, in pixels


Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, 14);

//https://arduinojson.org/v6/assistant/#/


// Configura el cliente NTP UDP
unsigned long utcOffsetInSeconds = 3600;
unsigned long NTP_INTERVAL = 60000;
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "europe.pool.ntp.org", utcOffsetInSeconds, NTP_INTERVAL);//europe.pool.ntp.org
// Horario Europa Central
TimeChangeRule CEST = {"CEST", Last, Sun, Mar, 2, 60};     // Hora de Verano de Europa Central
TimeChangeRule CET = {"CET ", Last, Sun, Oct, 3, 0};      // Hora Estandar de Europa Central
Timezone CE(CEST, CET);

double pasta[48];
double pastatrampolin[24];
bool fallo=0;
bool start=1;
double total=0;
double maximo=0;
double minimo=0;

int dia, mes, ano;
int hora ,minuto, segundo;
int oldminuto;
String fecha;
String hoy;
int valor;
unsigned long troceador;
int largura;
double medidor;
bool nuevodia=0;
bool bugui=0;



void connectWifi()
{

  // Connectando Wi-Fi
  wifiMulti.addAP("SSID_1", "PASSWORD_1");
  wifiMulti.addAP("SSID_2", "PASSWORD_2");
  wifiMulti.addAP("SSID_3", "PASSWORD_3");
 

  Serial.println("Pillando Wifi...");

  display.clearDisplay();
  display.setTextColor(WHITE);
  display.setTextSize(1);
  display.setCursor(0,10);
  display.print("   PILLANDO WIFI");
  display.display();



   while(wifiMulti.run() != WL_CONNECTED)
   {
      display.print(".");
      display.display();
      delay (1000);
   }


   if (wifiMulti.run() == WL_CONNECTED)
    {
      display.clearDisplay();
      display.setTextColor(WHITE);
      display.setTextSize(1);
      display.setCursor(20,10);
      display.print("WIFI PILLADA");
      display.setCursor(20,20);
      display.println (WiFi.localIP());
      display.setCursor(20,30);
      display.print("SSID: ");
      display.println (WiFi.SSID());
      display.setCursor(20,40);
      display.print("NIVEL: ");
      display.println (WiFi.RSSI());
      display.display();

    }


}

void vumetro(int largo)
{
  display.drawLine(0,57,largo,57,WHITE);
  display.drawLine(0,58,largo,58,WHITE);
  display.drawLine(0,59,largo,59,WHITE);
  display.drawLine(0,60,largo,60,WHITE);
  display.drawLine(0,61,largo,61,WHITE);
  display.drawLine(0,62,largo,62,WHITE);
  display.drawLine(0,63,largo,63,WHITE);
  display.drawLine(0,63,127,63,WHITE);
  display.drawPixel(32,62,WHITE);
  display.drawPixel(64,62,WHITE);
  display.drawPixel(96,62,WHITE);
  display.drawPixel(127,62,WHITE);
  display.drawPixel(32,61,WHITE);
  display.drawPixel(64,61,WHITE);
  display.drawPixel(96,61,WHITE);
  display.drawPixel(127,61,WHITE);
  display.display();
}
void visualizador(int puntero)
{

      display.setTextSize(3);
      display.setCursor(5,15);
      display.print(pasta[puntero]/10,2);
      display.setTextSize(1);
      display.print(" cts");
      display.setCursor(100,30);
      display.print("Kw/h");
      display.setCursor(0,45);
      display.print("min.");
      display.print(minimo/10,2);
      display.setCursor(69,45);
      display.print("max.");
      display.print(maximo/10,2);
      medidor=128/(maximo/10);
}

float naiveToFloat(const char *charArray) 
{
   long dataReal = 0;
   long dataDecimal = 0;
   long dataPow = 1;
   bool isNegative = false;
   if (*charArray == '-')
   {
      isNegative = true;
      ++charArray;
   }
   while ((*charArray >= '0' && *charArray <= '9'))
   {
      dataReal = (dataReal * 10) + (*charArray - '0');
      ++charArray;
   }
   if (*charArray == '.' || *charArray == ',' )
   {
      ++charArray;
      while ((*charArray >= '0' && *charArray <= '9'))
      {
         dataDecimal = (dataDecimal * 10) + (*charArray - '0');
         dataPow *= 10;
         ++charArray;
      }
   }
   float data = (float)dataReal + (float)dataDecimal / dataPow;
   return isNegative ? -data : data;
}



void getHttp()
{
 
 maximo=0;
 minimo=1000;
 fallo=0;  
 const String endpoint ="https://api.esios.ree.es/archives/70/download_json?locale=es";
 HTTPClient http;
 String response;
 http.begin(endpoint); //Specify the URL
 http.GET();
 response=http.getString();
 http.end();
 DynamicJsonDocument doc(16384);
 DeserializationError error = deserializeJson(doc, response);

if (error) 
{
  Serial.print("deserializeJson() fallo: ");
  Serial.println(error.c_str());
  fallo=1;
  return;
}
  int p=0;
 for (JsonObject PVPC_item : doc["PVPC"].as<JsonArray>()) 
{
  const char* PVPC_item_Dia = PVPC_item["Dia"]; // "02/02/2023", "02/02/2023", "02/02/2023", "02/02/2023", ...
  fecha=PVPC_item_Dia;  // fecha de los precios para cada hora
  //fecha.remove(2, 1);
  //fecha.remove(4, 1);
  const char* PVPC_item_PCB = PVPC_item["PCB"]; // "162,21", "167,26", "164,00", "164,89", "160,44", ...
  pastatrampolin[p] = naiveToFloat(PVPC_item_PCB);
  //Serial.println(pastatrampolin[p]);
  p++;
}
 
 for (int i = 0 ; i < 24 ;i++)
 {
    if (maximo<pastatrampolin[i]) maximo= pastatrampolin[i];
    if (minimo>pastatrampolin[i]) minimo= pastatrampolin[i];
    
 }
 
}


void setup()
{
  
    Serial.begin(115200);
  //initialize OLED
  Wire.begin(OLED_SDA, OLED_SCL);
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3c, false, false)) { // Address 0x3C for 128x32
    Serial.println(F("SSD1306 allocation failed"));
    for(;;); // Don't proceed, loop forever
  }
   pinMode(LED_BUILTIN,OUTPUT);
   pinMode(16,INPUT);
   pinMode(17,INPUT);
  
  display.clearDisplay();
  display.setTextColor(WHITE);
  display.setTextSize(3);
  display.setCursor(30,20);
  display.print("PVPC");
  display.display();
  delay(2000);
 connectWifi();
 delay(2000);
//******************************************************************************************
timeClient.begin(); // inicializo el timeclient despues de estar conectado a internet
//******************************************************************************************

  do
  {   
    // Obtengo la hora
    timeClient.update();
    //*************************************************************************
    unsigned long utc =  timeClient.getEpochTime();
    // Convertir marca de tiempo UTC UNIX a hora local
    time_t t = CE.toLocal(utc);  // y tomo nota de la hora actual

    hora = hour(t);
    minuto= minute(t);
    segundo= second(t);
 
    dia= day(t);
    mes= month(t);
    ano= year(t);
 
  } while (ano==1970);

  //Serial.println();
  //Serial.print("hora: ");
  valor = hora;
  //Serial.println(valor);

do 
  {
   getHttp();
   delay(2000);
   }while(fallo==1); 
fallo=0;
for (int i = 0; i < 24; i++)
   {
    pasta[i]=pastatrampolin[i];           
   }


}

void loop()
{
   if (wifiMulti.run() != WL_CONNECTED)
   {
     display.clearDisplay();
     display.setTextColor(WHITE);
     display.setCursor(20,20);
     display.print("WIFI NO CONECTADA");
     display.display();
     delay(1000);
   }

        
   if(((millis()-troceador)>=60000)||start)  // ejecuto cada 1 minuto
    {
    
    start=0;

    // Obtengo la hora
    timeClient.update();
    //*************************************************************************
    unsigned long utc =  timeClient.getEpochTime();
    // Convertir marca de tiempo UTC UNIX a hora local
    time_t t = CE.toLocal(utc);  // y tomo nota de la hora actual
    hora = hour(t);
    minuto= minute(t);
    segundo= second(t);
    dia= day(t);
    mes= month(t);
    ano= year(t);

   
    if (dia<10) 
      {
        hoy="0"+String(dia);
      }else
      {
        hoy=String(dia);
      }
      hoy+="/";
    if (mes<10) 
      {
        hoy+="0"+String(mes);
      }else
      {
        hoy+=String(mes);
      }
      hoy+="/";
      ano= year(t);
      hoy+=String(ano);  // la variable hoy tiene este formato    03/02/2023
  
     valor = hora;       // valor es la hora actual

     //******************************************************************************
     //  a partir de las 20:15 se publican los precios para el dia siguiente
     //******************************************************************************
     if(hora==20&&minuto>=30&&!nuevodia) 
      {
        do 
         {
          getHttp();
          delay(2000);
         }while(fallo==1); 
         fallo=0;
        
        if(hoy!=fecha)
        {
          for (int i = 0; i < 24; i++)
          {
            pasta[24+i]=pastatrampolin[i];     
          } 
            nuevodia=1; 
            bugui=1; 
        }
        else
        {
         for (int i = 0; i < 24; i++)
          {
            pasta[i]=pastatrampolin[i];     
          }
          bugui=0;
        }
        
      }
  
     //******************************************************************************
     //  a partir de las 00:00  copio los precios sobre los 24 valores iniciales y borro los 24 siguientes
     //****************************************************************************** 
     if (hora==0&&bugui)    
     {
      nuevodia=0;
      bugui=0;
     for (int i = 0; i < 24; i++)
          {
            pasta[i]=pasta[24+i];         
            pasta[24+i]=0;  
          }

     }

      display.clearDisplay();
      display.setTextColor(WHITE);
      display.setTextSize(1);
      display.setCursor(0,0);
      display.print(hora);
      display.print(":");
      if(minuto<10)display.print("0");
      display.print(minuto);
      display.print("      ");
      display.print(hoy);
      visualizador(hora);
      largura=(medidor*(pasta[valor]/10))+1;
      vumetro(largura);
      display.display();

       troceador=millis();

  }

  if (!digitalRead(17)) 
  { 
    hora++;
    if (hora>47) hora=47;
    display.clearDisplay();
    display.setTextColor(WHITE);
    display.setTextSize(1);
    display.setCursor(20,0);
 
    if(hora<24) display.print(hora);
    if(hora>=24) display.print(hora-24);
    display.print(":00 - ");
    if(hora<24) display.print(hora+1);
    if(hora>=24) display.print(hora-23);
    display.print(":00 ");
    
    visualizador(hora);
     largura=(medidor*(pasta[hora]/10))+1;
    vumetro(largura);
    troceador=millis()-56000;
    delay(200);
   }
  

  if (!digitalRead(16)) 
  {
    hora--;
    if (hora<0) hora=0;
    //if (hora<valor) hora=valor;
    display.clearDisplay();
    display.setTextColor(WHITE);
    display.setTextSize(1);
    display.setCursor(20,0);
    if(hora<24) display.print(hora);
    if(hora>=24) display.print(hora-24);
    display.print(":00 - ");
    if(hora<24) display.print(hora+1);
    if(hora>=24) display.print(hora-23);
    display.print(":00 ");
    visualizador(hora);
    largura=(medidor*(pasta[hora]/10))+1;
    vumetro(largura);
    troceador=millis()-56000;
    delay(200);
  }
   


}

