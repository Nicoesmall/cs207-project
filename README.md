
# CS207 Theremin-project
Arduino based Theremin

## How it works
The Arduino based Theremin is a ultrasonic frequency  which generates a radio wave and  gives out the frequency signal to the Arduino board.
An ultrasonic range finder connected to the Arduino board, and the code provides the Theremin effect when a person or some conductive material is placed next to the ultrasonic range finder.
This leads to a slight frequency deviation of the frequency which is registered by the Arduino software.
The Arduino acts in this case as a accurate frequency meter which transforms this frequency deviation into sound or control signals. 

## Installation Instructions
This is where I will tell you how to install the .ino code -- it's pretty straightforward.

### Menu Functions Code
/* Ping))) Sensor
  
   This sketch reads a PING))) ultrasonic rangefinder and returns the
   distance to the closest object in range. To do this, it sends a pulse
   to the sensor to initiate a reading, then listens for a pulse 
   to return.  The length of the returning pulse is proportional to 
   the distance of the object from the sensor.
     
   The circuit:
	* +V connection of the PING))) attached to +5V
	* GND connection of the PING))) attached to ground
	* SIG connection of the PING))) attached to digital pin 7
  
   */

// this constant won't change.  It's the pin number
// of the sensor's output:
const int pingPin = 7;
const int buzzPin = 9;

void setup() {
  // initialize serial communication:
  Serial.begin(9600);
  pinMode(buzzPin,OUTPUT);
}

void loop()
{
  // establish variables for duration of the ping, 
  // and the distance result in inches and centimeters:
  long duration, inches, cm;

  // The PING))) is triggered by a HIGH pulse of 2 or more microseconds.
  // Give a short LOW pulse beforehand to ensure a clean HIGH pulse:
  pinMode(pingPin, OUTPUT);
  digitalWrite(pingPin, LOW);
  delayMicroseconds(2);
  digitalWrite(pingPin, HIGH);
  delayMicroseconds(5);
  digitalWrite(pingPin, LOW);
    // The same pin is used to read the signal from the PING))): a HIGH
  // pulse whose duration is the time (in microseconds) from the sending
  // of the ping to the reception of its echo off of an object.
  pinMode(pingPin, INPUT);
  duration = pulseIn(pingPin, HIGH);

  // convert the time into a distance
  inches = microsecondsToInches(duration);
  cm = microsecondsToCentimeters(duration);
  int freq = map(cm,0,14,200,500);
  tone(buzzPin, freq);
  Serial.print(inches);
  Serial.print("in, ");
  Serial.print(cm);
  Serial.print("cm");
  Serial.println();
  
  delay(0);
}

long microsecondsToInches(long microseconds)
{
}

long microsecondsToCentimeters(long microseconds)
{
}


## Manifest
* README.md -- this file
* arduino based theremin user's manual
* img -- directory where images are stored

## Copyright Notice
This project is liscensed under the MIT liscense. 

## Credits / Aknowledgements 

   http://www.arduino.cc/en/Tutorial/Ping
   
   http://www.arduino.cc/en/Tutorial/TonePitchFollower?from=Tutorial.Tone2

## Contact
Email jadelov416@gmail.com

## Bugs List

* Needs more variety on tones and some upgrades.
  
  
