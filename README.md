# cs207-project
NaV-1 Arduino Synthesizer

## Configuration Instructions
This is where I will tell you how I built my project -- including pictures and whatnot!

## Installation Instructions
This is where I will tell you how to install the .ino code -- it's pretty straightforward.

### Menu Functions
void menuDraw(int cursX, int cursY){
  lcd.setCursor (cursorX, cursorY);
  lcd.print(" "); //Erase Old Cursor Position
  cursorX = cursX;
  cursorY = cursY;
  lcd.setCursor (cursorX,cursorY);
  lcd.print(">"); //Draw New Cursor Position
}

void menuRef(String line1, String line2, String line3, String line4){
  lcd.clear();
  lcd.print(line1);
  lcd.setCursor(0,1);
  lcd.print(line2);
  lcd.setCursor(0,2);
  lcd.print(line3);
  lcd.setCursor(0,3);
  lcd.print(line4);
}

void menuCursorSet(int x, int y){
  cursorX = x;
  cursorY = y;
}

byte readOneByte(byte arg){
  byte inByte = 0;
  soundgin.write(0x1B);
  soundgin.write((byte)0x00);
  soundgin.write(arg);

  while (soundgin.available()){
    inByte = soundgin.read();
    //lcd.print(inByte);
  }
  return inByte;
}

void writeOneMask(byte location, byte mask, byte arg){
  soundgin.write(0x1B);
  soundgin.write(0x04);
  soundgin.write(location);
  soundgin.write(mask);
  soundgin.write(arg);

  arg = mask & arg;
  mask = ~mask;
  byte result = mask & patch[location];
  patch[location] = arg | result;
}

void writeOneByte(byte location, byte arg){
  soundgin.write(0x1B);
  soundgin.write(0x01);
  soundgin.write(location);
  soundgin.write(arg);

  patch[location] = arg;
}


void lcdPrint (int x, int y, char text[]){
  lcd.setCursor(x,y);
  lcd.print(text);
}

void waveTypePrint (byte index){

  waveform = index; //global

  lcd.setCursor (6,1);
  switch(index){
  case 0:
    lcd.print(F("Sine    "));
    break;
  case 1:
    lcd.print(F("Triangle"));
    break;
  case 2:
    lcd.print(F("Saw     "));
    break;
  case 3:
    lcd.print(F("Ramp    "));
    break;
  case 4:
    lcd.print(F("Pulse   "));
    break;
  case 5:
    lcd.print(F("Noise   "));
    break;
  case 6:
    lcd.print(F("Level   "));
    break;
  case 7:
    lcd.print(F("Vocal   "));
    break;
  }
}

void printStar (byte x, byte y, byte star){
  lcd.setCursor (x,y);
  if (star){
    lcd.print("*");
  }
  else {
    lcd.print(" ");
  }
}

void printCursor (byte cursX, byte cursY, byte cursorType){
  lcd.setCursor (cursorX, cursorY);
  lcd.print(" "); //Erase Old Cursor Position
  cursorX = cursX;
  cursorY = cursY;
  lcd.setCursor (cursorX,cursorY);
  if (cursorType == 1){
    lcd.print("^");
  }
  else if (cursorType == 2){
    lcd.print("-");
  }
  else if (cursorType == 3){
    lcd.print(">");
  }
  else if (cursorType == 4){
    lcd.print(":");
  }
}

void menuChange(byte level){
  menuLevelState = level;
  menuState = 1;
  eventType = REFRESH;
  menuLevelSwitch();

}
## Eeprom info

void patchWrite(byte patchNum)
{

  byte i; //Loop Counter
  byte data = 0; //Array Index
  unsigned int address = (patchNum * 192);
  // Write Soundgin data
  do{
    Wire.beginTransmission(EEPROM1);
    Wire.write((int)((address) >> 8));   // High Byte
    Wire.write((int)((address) & 0xFF)); // Low Byte
    for (i=0; i < 16; i++){ // Write 16 Byte block of data
      Wire.write(patch[data]);
      data++;
    }
    Wire.endTransmission();
    address = address + 16; // Move address to next 16 Byte block
    delay(10);
  }
  while (data < 144); // Keep writing 16 Byte blocks untill all 144 registers are sent
  // Write Patch name
  Wire.beginTransmission(EEPROM1);
  Wire.write((int)((address) >> 8));   // High Byte
  Wire.write((int)((address) & 0xFF)); // Low Byte
  for (i=0; i < 16; i++){
    Wire.write((byte) patchName[i]);
  }
  Wire.endTransmission();
  delay(10);
}

void burnPatch(){ 
  byte param = 0;
  do{
    if (digitalRead(CTS)==LOW){
      writeOneByte(param, patch[param]);
      param++;
    }
  }
  while(param < 128);
  release(); 
}

//TEST MEMFREE
// function to return the amount of free RAM
int memoryFree()
{
  int freeValue;
  if((int)__brkval == 0)
    freeValue = ((int)&freeValue) - ((int)&__bss_end);
  else
    freeValue = ((int)&freeValue) - ((int)__brkval);
  return freeValue;
}

void patchRead(byte patchNum)
{
  byte data = 0; // Array Index
  byte i; // Loop Counter
  unsigned int address = (patchNum * 192);

  for (i=0; i < 9; i++){ // Read 9 Blocks of 16 Bytes = 144 Bytes
    Wire.beginTransmission(EEPROM1);
    Wire.write((int)(address >> 8)); // High Byte
    Wire.write((int)(address & 0xFF)); // Low Byte
    Wire.endTransmission();

    Wire.requestFrom(EEPROM1, 16);
    while(Wire.available())
    {
      patch[data] = Wire.read();
      data++;
    }
    address = address + 16;
    delay(10);
  }
  //Read Patch Name
  data = 0;
  Wire.beginTransmission(EEPROM1);
  Wire.write((int)(address >> 8)); // High Byte
  Wire.write((int)(address & 0xFF)); // Low Byte
  Wire.endTransmission();

  Wire.requestFrom(EEPROM1, 16);
  while(Wire.available()) 
  {
    patchName[data] = Wire.read();
    data++;
  }
}


void eepromInit(){
  boolean init = true;
  byte i = 0;
  for (i = 0; i < 200; i++){
    if ((digitalRead(BUTTON_ENTER) == HIGH) | (digitalRead(BUTTON_EXIT) == HIGH)) init = false;
    delay(10);
  }

  if (init == true){
    lcd.clear();
    lcd.print(F("Initializing EEPROM"));
    patchWrite(0);
    for (i = 0; i < 15; i++){
      patchName[i] = 32;
    }
    patch[16] = 144;

    for (i = 1; i < 100; i++){
      lcd.setCursor(8,2);
      lcd.print("%");
      lcd.print(i + 1);
      patchWrite(i);
    }
  }
}



## Manifest
* README.md -- this file
* hello.ino -- the hello world "blink" file for testing
* img -- directory where images are stored

## Copyright Notice
This project is liscensed under the MIT liscense. 

## Credits / Aknowledgements 
This is where I will tell you where I got source code from. 
Thanks to (...) for helping...

## Contact
If you want to contribute to this project, feel free to email me at ...

## Bugs List
The following problems need to be addressed: 
* The whole thing. 
![Canadian Flag](/img/Canada-Flag-5.jpg)


