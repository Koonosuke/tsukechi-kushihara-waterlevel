/*
  *@File  : DFRobot_Distance_A02.ino 
  *@Brief : This example use A02YYUW ultrasonic sensor to measure distance
  *         With initialization completed, We can get distance value 
  *@Copyright [DFRobot](https://www.dfrobot.com),2016         
  *           GUN Lesser General Pulic License
  *@version V1.0           
  *@data  2019-8-28
*/
#include <Arduino.h>
#include <SPI.h>
#include <Wire.h>

// Need this for the lower level access to set them up.
#include <HardwareSerial.h>

//Define two Serial devices mapped to the two internal UARTs
HardwareSerial MySerial0(0);
HardwareSerial MySerial1(1);
unsigned char data[4]={};
float distance;

void setup()
{
 Serial.begin(57600);
// Configure MySerial0 on pins TX=6 and RX=7 (-1, -1 means use the default)
 MySerial0.begin(9600, SERIAL_8N1, -1, -1);
 MySerial1.begin(9600, SERIAL_8N1, 9, 10);
 MySerial0.print("MySerial0");
}

void loop()
{
    Serial.print("distance=");
    do{
     for(int i=0;i<4;i++)
     {
       data[i]=MySerial1.read();
     }
  }while(MySerial1.read()==0xff);

  MySerial1.flush();

  if(data[0]==0xff)
    {
      int sum;
      sum=(data[0]+data[1]+data[2])&0x00FF;
      if(sum==data[3])
      {
        distance=(data[1]<<8)+data[2];
        if(distance>30)
          {
           Serial.print("distance=");
           Serial.print(distance/10);
           Serial.println("cm");
          }else 
             {
               Serial.println("Below the lower limit");
             }
      }else Serial.println("ERROR");
     }
     delay(100);
}