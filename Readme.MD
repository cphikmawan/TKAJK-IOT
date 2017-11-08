# IoT ---- Detektor Air Hangat
### Komponen Yang Diperlukan

1. Arduino Uno
1. Servo (2 Buah)
1. Kabel jumper
1. Sensor Suhu ds18b20
1. Modul wifi
1. Breadboard
1. Resistor 4,7k

### Software Pendukung
1. Arduino Software
1. Mobile App (Blynks)

### Library
1. One wire
1. Blynks

### Mekanisme IoT
1. 10 Menit pertama bak air akan diisi oleh air dingin yang digerakkan oleh servo air dingin.
1. Kemudian servo air panas akan otomatis terbuka dan servo air dingin akan tertutup.
1. Jika sensor suhu sudah mendeteksi suhu air sesuai dengan yang diinginkan maka servo air panas akan menutup secara otomatis. 

### Rangkaian
![GitHub Logo](/images/Arduino.png)

### Code
```c
#include <OneWire.h>
#include <Servo.h>
Servo servoDingin,servoPanas;
// OneWire DS18S20, DS18B20, DS1822 Temperature Example
//
// http://www.pjrc.com/teensy/td_libs_OneWire.html
//
// The DallasTemperature library can do all this work for you!
// http://milesburton.com/Dallas_Temperature_Control_Library

OneWire  ds(2);  // on pin 2 (a 4.7K resistor is necessary)
int dingin = 1;

void setup(void) {
  Serial.begin(9600);
  
  servoDingin.attach(10);
  servoDingin.write(90);
  servoPanas.attach(12);
  servoPanas.write(0);
}

void loop(void) {
  byte i;
  byte present = 0;
  byte type_s;
  byte data[12];
  byte addr[8];
  float celsius, fahrenheit;

  if ( !ds.search(addr)) {
    Serial.println("No more addresses.");
    Serial.println();
    ds.reset_search();
    delay(250);
    return;
  }

  Serial.print("ROM =");
  for( i = 0; i < 8; i++) {
    Serial.write(' ');
    Serial.print(addr[i], HEX);
  }

  if (OneWire::crc8(addr, 7) != addr[7]) {
      Serial.println("CRC is not valid!");
      return;
  }
  Serial.println();

  // the first ROM byte indicates which chip
  switch (addr[0]) {
    case 0x10:
      Serial.println("  Chip = DS18S20");  // or old DS1820
      type_s = 1;
      break;
    case 0x28:
      Serial.println("  Chip = DS18B20");
      type_s = 0;
      break;
    case 0x22:
      Serial.println("  Chip = DS1822");
      type_s = 0;
      break;
    default:
      Serial.println("Device is not a DS18x20 family device.");
      return;
  } 

  ds.reset();
  ds.select(addr);
  ds.write(0x44);        // start conversion, use ds.write(0x44,1) with parasite power on at the end

  delay(1000);     // maybe 750ms is enough, maybe not
  // we might do a ds.depower() here, but the reset will take care of it.

  present = ds.reset();
  ds.select(addr);    
  ds.write(0xBE);         // Read Scratchpad

  Serial.print("  Data = ");
  Serial.print(present, HEX);
  Serial.print(" ");
  for ( i = 0; i < 9; i++) {           // we need 9 bytes
    data[i] = ds.read();
    Serial.print(data[i], HEX);
    Serial.print(" ");
  }
  Serial.print(" CRC=");
  Serial.print(OneWire::crc8(data, 8), HEX);
  Serial.println();

  // Convert the data to actual temperature
  // because the result is a 16 bit signed integer, it should
  // be stored to an "int16_t" type, which is always 16 bits
  // even when compiled on a 32 bit processor.
  int16_t raw = (data[1] << 8) | data[0];
  if (type_s) {
    raw = raw << 3; // 9 bit resolution default
    if (data[7] == 0x10) {
      // "count remain" gives full 12 bit resolution
      raw = (raw & 0xFFF0) + 12 - data[6];
    }
  } else {
    byte cfg = (data[4] & 0x60);
    // at lower res, the low bits are undefined, so let's zero them
    if (cfg == 0x00) raw = raw & ~7;  // 9 bit resolution, 93.75 ms
    else if (cfg == 0x20) raw = raw & ~3; // 10 bit res, 187.5 ms
    else if (cfg == 0x40) raw = raw & ~1; // 11 bit res, 375 ms
    //// default is 12 bit resolution, 750 ms conversion time
  }
  celsius = (float)raw / 16.0;
  fahrenheit = celsius * 1.8 + 32.0;
  Serial.print("  Temperature = ");
  Serial.print(celsius);
  Serial.print(" Celsius, ");
  Serial.print(fahrenheit);
  Serial.println(" Fahrenheit");
  printf("%d",dingin);
  if (dingin){
    delay(5000);
    servoDingin.write(0);
    servoPanas.write(90);
    dingin=0;
  }
  if(celsius > 30) {
    servoPanas.write(0);
    servoDingin.write(0);
  }
}
```

#### Referensi
* http://www.pjrc.com/teensy/td_libs_OneWire.html
* http://milesburton.com/Dallas_Temperature_Control_Library