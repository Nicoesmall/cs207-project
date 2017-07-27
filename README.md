# cs207-project
NaV-1 Arduino Synthesizer

## Configuration Instructions
This is where I will tell you how I built my project -- including pictures and whatnot!

## Installation Instructions
This is where I will tell you how to install the .ino code -- it's pretty straightforward.

### Menu Functions Code
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
## Raw.ino
/**********************************************************************
 *                 NaV-1 Arduino Soundgin Synth V1.0                  *
 *               Visit - notesandvolts.com for tutorial               *
 *                                                                    *
 *               Requires Arduino Version 1.0 or Later                *
 **********************************************************************/

#include <SoftwareSerial.h>
#include <MIDI.h>
#include <LiquidCrystal.h>
#include <Wire.h>
#include <avr/pgmspace.h>

#define CTS 2 
#define txPin 3
#define rxPin 4
#define ROTARY_PIN1 5
#define ROTARY_PIN2 7
#define LCD_RS 6
#define LCD_EN 8
#define LCD_DB4 A0
#define LCD_DB5 A1
#define LCD_DB6 A2
#define LCD_DB7 A3
#define LED 13
#define BUTTON_ENTER 9
#define BUTTON_EXIT 10

#define NumNotes 96

#define TIMEOUT 300
#define DIR_CCW 0x10
#define DIR_CW 0x20

#define ENCODER 1
#define ENTER 2
#define EXIT 3
#define REFRESH 4
#define NONE 5

#define PATCHMENU 0
#define MAINMENU 1
#define OSCMENU 2
#define OSCWAVEMENU 3
#define PATCHNAMEMENU 5
#define PATCHSAVE 99
#define RAW 6
#define ADSR 7
#define EEPROM1 0x50 //Address of eeprom chip

#define OSCA1 0x10
#define OSCA2 0x20
#define OSCA3 0x30
#define OSCB1 0x50
#define OSCB2 0x60
#define OSCB3 0x70


// Create a SoftwareSerial object named "Soundgin"
SoftwareSerial soundgin = SoftwareSerial(rxPin, txPin);

// Initialize the LCD and Assign pins
LiquidCrystal lcd(LCD_RS, LCD_EN, LCD_DB4, LCD_DB5, LCD_DB6, LCD_DB7);

//TEST ONLY
// variables created by the build process when compiling the sketch
extern int __bss_end;//Memory test
extern void *__brkval;//Memory test
// Global Variables
byte currentNote = 0;
byte newNote = 0;
volatile unsigned char state = 0;
boolean enterPress = false;
boolean exitPress = false;
unsigned long enterTime;
unsigned long exitTime;
int myCursor = 10;

int menuState = 1;
int menuLevelState = 0; //Level of Menu
//boolean menuRefresh = true;
int cursorX = 0;
int cursorY = 0;
byte editMode = 0; //Used by name func to edit letters

int backState = 1;
byte eventType = 0;
boolean encoderDirection; // True for CW. False for CCW
byte oscSelect = 0; // Stores selected Oscillator
String oscName;
byte oscOffset = 0; //Used by RAW EDIT
byte oscStatus = 0; // Stores Osc status byte
byte waveform = 0; // Stores wave type
byte oldWaveform = 0; // old wave for restore

// Array of Patch parameters
byte patchNumber = 0;
byte patch[128] = {//Holds 128 Byte Soundgin Parameters
  7,0,0,0,0,0,0,0,127,0,0,0,0,0,0,0, //Mix A
  146,0,0,0,0,0,0,0,0,0,0,0,0,240,204,9, //Osc A1
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc A2
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc A3
  7,0,0,0,0,0,0,0,127,0,0,0,0,0,0,0, //Mix B
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc B1
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc B2
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc B3
};
char patchName[16] = "Notes and Volts";//Holds patch name

// Lookup Table for Rotary Encoder function
const unsigned char relookup[7][4] PROGMEM = {
  {
    0x0, 0x2, 0x4,  0x0    }
  , 
  {
    0x3, 0x0, 0x1, 0x10    }
  ,
  {
    0x3, 0x2, 0x0,  0x0    }
  , 
  {
    0x3, 0x2, 0x1,  0x0    }
  ,
  {
    0x6, 0x0, 0x4,  0x0    }
  , 
  {
    0x6, 0x5, 0x0, 0x20    }
  ,
  {
    0x6, 0x5, 0x4,  0x0    }
  ,
};

// Lookup Table - Coverts incoming Midi Note Number to Soundgin note number
const byte lookup[NumNotes] PROGMEM = {
  0,1,2,3,4,5,6,7,8,9,10,11,
  16,17,18,19,20,21,22,23,24,25,26,27,
  32,33,34,35,36,37,38,39,40,41,42,43,
  48,49,50,51,52,53,54,55,56,57,58,59,
  64,65,66,67,68,69,70,71,72,73,74,75,
  80,81,82,83,84,85,86,87,88,89,90,91,
  96,97,98,99,100,101,102,103,104,105,106,107,
  112,113,114,115,116,117,118,119,120,121,122,123
};

// Soundgin Initialization Commands
// **27 = Command Charater (Precedes all commands)**
// **27,1,Memory,Param (Stores Parameter in Memory Location)**
byte sgInit[] PROGMEM = {
  27,6, // (6 = Clear Mixers A & B)
  27,84, 27,85, 27,86, // (84,85,86 = Release Osc A1,A2,A3)
  27,116, 27,117, 27,118,  // (116,117,118 = Release Osc B1,B2,B3)
  27,1,136,127, // (136 = Set Master Volume to 127 - FULL)
  27,1,8,127, // (8 = Set Mixer A Volume to 127 - FULL)
  27,1,72,127, // (72 = Set Mixer B Volume to 127 - FULL)
  27,1,0,1, // Send Osc A1 to Mixer A
  27,1,1,0, // Turn Off PWM for Osc A1
  27,1,16,146, // 16 = Osc A Control = OvF Mode on, Saw Wave set
  27,1,29,240 ,27,1,30,204, 27,1,31,9 };// 29,30,31 = Osc A1 Atack,Decay,Release


//Lookup Table ADSR Values
const unsigned int adsrVal[] PROGMEM = {
  2,6,8,24,16,48,24,72,38,114,56,168,68,204,80,240,100,300,
  250,750,500,1500,800,2400,1000,3000,2800,8400,5600,16800,11200,33600
};

void setup()
{
  //Set Arduino Pins
  pinMode (LED, OUTPUT);
  pinMode (rxPin, INPUT);
  pinMode (txPin, OUTPUT);
  pinMode (CTS, INPUT);

  pinMode(ROTARY_PIN1, INPUT); //Set Rotary Encoder Pin 1 to Input
  digitalWrite(ROTARY_PIN1, HIGH); //Enable on board resistor
  pinMode(ROTARY_PIN2, INPUT);
  digitalWrite(ROTARY_PIN2, HIGH);

  pinMode(BUTTON_ENTER,INPUT);            // default mode is INPUT
  digitalWrite(BUTTON_ENTER, HIGH);     // Turn on the internal pull-up resistor
  pinMode(BUTTON_EXIT,INPUT);            // default mode is INPUT
  digitalWrite(BUTTON_EXIT, HIGH);     // Turn on the internal pull-up resistor

  Wire.begin();
  soundgin.begin(9600); //Set Software Serial Baud Rate to 9600
  //initPatch(); //Run patch function - Initializes and sets up sound on Soundgin Chip
  MIDI.begin(MIDI_CHANNEL_OMNI); // Initialize the Midi Library.
  MIDI.setHandleNoteOn(MyHandleNoteOn); //set MyHandleNoteOn as the callback function

    // Print to LCD
  lcd.begin(20, 4);
  delay(500);
  lcd.print(F(" NOTESANDVOLTS.COM "));
  lcd.print(F("         1.0 "));
  eepromInit();
  initPatch(); //Run patch function - Initializes and sets up sound on Soundgin Chip
  menuChange(PATCHMENU);
  eventType = REFRESH;
}

void loop() // Main Loop
{
  MIDI.read(); // Continually check RX for incoming Midi Commands
  menu(); // Check state of Rotary Encoder and Buttons - Update LCD
}

## Raw.ino
// Raw Edit Mode - **For development only**
void rawEdit(){ 

  if (eventType == REFRESH){
    //oscStatus = readOneByte(oscSelect);
    menuState = 0;
    editMode = 0;
    oscStatus = patch[oscSelect];
    menuCursorSet(4,0);
    menuRef(oscName, "", "", "");
    menuDraw(4,0);
    for (byte i = 0; i < 16; i++){
      rawCurs(i,3);
    }
    rawCurs(0,3);  
    eventType = NONE;
  }

  else if (eventType == ENCODER){
    if (editMode == 0){
      if (menuState < 0) menuState = 15;
      if (menuState > 15) menuState = 0;
      rawCurs(menuState,3);
    }
    else {
      if (encoderDirection == true){
        menuState -= 1; //keep menuState from changing
        patch[menuState + oscOffset]++;
        writeOneByte(menuState + oscOffset, patch[menuState + oscOffset]);
        rawCurs(menuState,4);
      }
      if (encoderDirection == false){ // menuState from changing
        menuState += 1;
        patch[menuState + oscOffset]--;
        writeOneByte(menuState + oscOffset, patch[menuState + oscOffset]);
        rawCurs(menuState,4);
      }
    };
  }

  else if (eventType == ENTER){
    editMode = 1 - editMode; //Flips 1 to 0 and visa-versa
    if (editMode == 1) {
      rawCurs(menuState, 4);
      oldWaveform = patch[menuState + oscOffset];
    }
    else {
      rawCurs(menuState, 3);     
    }
  }

  else if (eventType == EXIT){
    if (editMode == 1){
      patch[menuState + oscOffset] = oldWaveform;
      writeOneByte(menuState + oscOffset, patch[menuState + oscOffset]);
      rawCurs(menuState, 3);
      editMode = 0;
    }
    else menuChange(MAINMENU);
  }
}

void rawCurs(int pos, byte cursType){

  lcd.setCursor(0,1);
  lcd.print("   ");
  lcd.setCursor(0,1);
  lcd.print(menuState + oscOffset);

  if (pos < 4) {
    printCursor(((pos*4))+4,0,cursType);
    lcd.print("   ");
    printCursor(((pos*4))+4,0,cursType);
    lcd.print(patch[pos + oscOffset]);
  }
  if (pos >= 4 && pos < 8) {
    printCursor(((pos-4)*4)+4,1,cursType);
    lcd.print("   ");
    printCursor(((pos-4)*4)+4,1,cursType);
    lcd.print(patch[pos + oscOffset]);
  }
  if (pos >= 8 && pos < 12){
    printCursor(((pos-8)*4)+4,2,cursType);
    lcd.print("   ");
    printCursor(((pos-8)*4)+4,2,cursType);
    lcd.print(patch[pos + oscOffset]);
  } 
  if (pos >= 12) {
    printCursor(((pos-12)*4)+4,3,cursType);
    lcd.print("   ");
    printCursor(((pos-12)*4)+4,3,cursType);
    lcd.print(patch[pos + oscOffset]);
  }
}

## Soundgin.ino
// Soundgin and MIDI functions

void initPatch() // Patch - Initialize Soundgin & set Osc A1 to square wave
{
  // 0 with 10 x 27 = Soundgin Hardware Reset
  soundgin.write(byte(0)); 
  soundgin.write(27);
  soundgin.write(27);
  soundgin.write(27);
  soundgin.write(27);
  soundgin.write(27);
  soundgin.write(27);
  soundgin.write(27);
  soundgin.write(27);
  soundgin.write(27);
  soundgin.write(27);
  delay (500); // Wait until soundgin is reset
  patchRead(0);
  burnPatch();
  release();
}

/* MyHandleNoteOn is called when The Midi Library receives
 a Midi note. This function uses a lookup table to convert
 the Midi note number to the correct Soundgin PlayNote Number.
 The currentNote variable holds the first key that is pressed.
 The newNote variable is used if another key is pressed while
 the original key is still being held. That way if the new note is 
 released, the original note is retriggered.
 */
void MyHandleNoteOn(byte channel, byte pitch, byte velocity) { 

  if (velocity != 0) { 
    digitalWrite(LED,HIGH);  //Turn LED on

    if (currentNote == 0) {
      currentNote = pitch;
      trigger(pgm_read_byte(&lookup[currentNote]));
    }
    else {
      newNote = pitch;
      trigger(pgm_read_byte(&lookup[newNote]));
    }
  }

  if (velocity == 0) {//A NOTE ON message with a velocity = Zero is actualy a NOTE OFF
    digitalWrite(LED,LOW);//Turn LED off
    if (pitch == newNote){
      newNote = 0;
      trigger(pgm_read_byte(&lookup[currentNote]));
    }
    else if (pitch == currentNote && newNote != 0){
      currentNote = newNote;
      newNote = 0;
      trigger(pgm_read_byte(&lookup[currentNote]));
    }
    else if (pitch == currentNote && newNote == 0){
      release();
      currentNote = 0;
      newNote = 0;
    }
  }
}

void trigger(byte note){

  soundgin.write(27); //Soundgin Command
  soundgin.write(88); //Osc A1
  soundgin.write(note);

  soundgin.write(27); //Soundgin Command
  soundgin.write(89); //Osc A2
  soundgin.write(note);

  soundgin.write(27); //Soundgin Command
  soundgin.write(90); //Osc A3
  soundgin.write(note);

  soundgin.write(27); //Soundgin Command
  soundgin.write(120); //Osc B1
  soundgin.write(note);

  soundgin.write(27); //Soundgin Command
  soundgin.write(121); //Osc B2
  soundgin.write(note);

  soundgin.write(27); //Soundgin Command
  soundgin.write(122); //Osc B3
  soundgin.write(note);

  //soundgin.write(27); //Soundgin Command
  //soundgin.write(83); //Mix A

  //soundgin.write(27); //Soundgin Command
  //soundgin.write(115); //Mix B
}


void release(){
  soundgin.write(27); //Soundgin command
  soundgin.write(84); //Osc A1

  soundgin.write(27); //Soundgin command
  soundgin.write(85); //Osc A2

  soundgin.write(27); //Soundgin command
  soundgin.write(86); //Osc A3

  soundgin.write(27); //Soundgin command
  soundgin.write(116); //Osc B1

  soundgin.write(27); //Soundgin command
  soundgin.write(117); //Osc B2

  soundgin.write(27); //Soundgin command
  soundgin.write(118); //Osc B3

  //soundgin.write(27); //Soundgin Command
  //soundgin.write(87); //Mix A

  //soundgin.write(27); //Soundgin Command
  //soundgin.write(119); //Mix B
}


## User_ Interface.ino
// User Interface Functions

// re_state function - Check for Rotary Encoder change
// Use relookup Table to check if encoder is in valid state
unsigned char re_state() {
  unsigned char pinstate = (digitalRead(ROTARY_PIN2) << 1) | digitalRead(ROTARY_PIN1);
  //state = relookup[state & 0xf][pinstate];
  state = pgm_read_byte(&relookup[state & 0xf][pinstate]);
  return (state & 0x30);
}

// menu function - Check state of Rotary Encoder and Buttons - Update LCD
void menu() {
  unsigned char result = re_state();

  if (result) {
    if (result == DIR_CCW){
      menuState--;
      eventType = ENCODER;
      encoderDirection = false;

      menuLevelSwitch();
    }
    else {
      menuState++;
      //menuMainSwitch();
      eventType = ENCODER;
      encoderDirection = true;
      menuLevelSwitch();
    }  

  }
  // Enter Button
  if ( digitalRead(BUTTON_ENTER) == LOW & enterPress == false ) {
    enterPress = true;
    enterTime = millis();
    eventType = ENTER;
    menuLevelSwitch();
  }
  else if ( digitalRead(BUTTON_ENTER) == HIGH & millis() > enterTime + TIMEOUT ) {
    enterPress = false;
  }
  // Exit Button
  if ( digitalRead(BUTTON_EXIT) == LOW & exitPress == false ) {
    exitPress = true;
    exitTime =millis();
    eventType = EXIT;
    menuLevelSwitch();
  }
  else if ( digitalRead(BUTTON_EXIT) == HIGH & millis() > exitTime + TIMEOUT ) {
    exitPress = false;
  }
}

void menuLevelSwitch(){
  switch(menuLevelState){
  case 0:
    menuPatch();
    break;
  case 1:
    menuMainSwitch();
    break;
  case 2:
    menuOscSwitch();
    break;
  case 3: 
    menuOscWaveform();
    break;
  case 4:
    waveformSelect();
    break;
  case 5:
    patchNameSet();
    break;
  case 6:
    rawEdit();
    break;
  case 7:
    adsr();
    break;
  case 99:
    savePatch();
    break;
  }
}



void menuPatch(){
  // Draw menu if required
  if (eventType == REFRESH){
    editMode = 1;
    menuCursorSet(0,0);
    menuRef("" , "", " Edit", "");
    lcd.setCursor(1,0);
    if (patchNumber < 10) lcd.print("0");
    lcd.print( patchNumber );
    lcd.print(":");
    lcd.print(patchName);
    printCursor(0,0,4);
    eventType = NONE;
  }
  if (editMode == 1) menuState = 1;

  switch(menuState){
  case 0:
    menuState = 2;
    menuDraw(0,2);
    break;
  case 1:
    if (eventType == ENCODER){ 

      if (editMode == 0) {
        menuDraw(0,0);
      }

      else{
        if (encoderDirection == true){
          patchNumber++;
        }
        else if (encoderDirection == false){
          //menuState = 1;
          patchNumber--;
        }
        if (patchNumber == 100) patchNumber = 0;
        else if (patchNumber == 255) patchNumber = 99;
        release();
        lcdPrint(1,0,"  ");
        lcd.setCursor(1,0);
        if (patchNumber < 10) lcd.print("0");
        lcd.print(patchNumber);
        lcd.print(":");
        patchRead(patchNumber);
        lcd.print(patchName);
        burnPatch();
        //eventType = REFRESH;
      }
    }

    else if (eventType == ENTER){
      editMode = 1 - editMode; //Flips 1 to 0 and visa-versa
      if (editMode == 1) {
        printCursor(0,0,4);
      }
      else {
        printCursor(0,0,3);    
      }
    }
    break; 
  case 2://Edit
    if (eventType == ENCODER){
      menuDraw(0,2);
    }
    else if (eventType == ENTER){
      menuChange(MAINMENU);
    }
    break;
  case 3:
    menuState = 1;
    menuDraw(0,0);//call menu drawing function
    break;
  }
}//End menuPatch

void menuMainSwitch(){

  // Draw menu if required
  if (eventType == REFRESH){
    menuCursorSet(3,0);
    //menuRef("Osc A1 A2 A3", "    B1 B2 B3", "Mix A B Ctrl", "    Trig Name");
    lcd.clear();
    lcd.print(F("Osc A1 A2 A3"));
    lcd.setCursor(0,1);
    lcd.print(F("    B1 B2 B3"));
    lcd.setCursor(0,2);
    lcd.print(F("Mix A B Ctrl"));
    lcd.setCursor(0,3);
    lcd.print(F("    Trig Name"));

    //lcd.setCursor (17,3);//TEST ONLY
    //lcd.print(memoryFree()); // print the free memory TEST ONLY

    menuDraw(3,0);
    eventType = NONE;
  }
  if (eventType == EXIT){
    menuChange(PATCHSAVE);
  }
  switch(menuState){
  case 0:
    menuState = 11;
    menuDraw(8,3);//call menu drawing function
    break;
  case 1:
    if (eventType == ENCODER){
      menuDraw(3,0);//call menu drawing function
    }
    else if (eventType == ENTER){
      oscSelect = OSCA1;
      oscName = "A1";
      oscOffset = 16;
      menuChange(OSCMENU);
    }
    else if (eventType == EXIT){

    }
    break;

  case 2:
    if (eventType == ENCODER){
      menuDraw(6,0);
    }
    else if (eventType == ENTER){
      oscSelect = OSCA2;
      oscName = "A2";
      oscOffset = 32;
      menuChange(OSCMENU);
    }
    break;
  case 3:
    if (eventType == ENCODER){
      menuDraw(9,0);
    }
    else if (eventType == ENTER){
      oscSelect = OSCA3;
      oscName = "A3";
      oscOffset = 48;
      menuChange(OSCMENU);
    }
    break;
  case 4:
    if (eventType == ENCODER){
      menuDraw(3,1);
    }
    else if (eventType == ENTER){
      oscSelect = OSCB1;
      oscName = "B1";
      oscOffset = 80;
      menuChange(OSCMENU);
    }
    break;
  case 5:
    if (eventType == ENCODER){
      menuDraw(6,1);
    }
    else if (eventType == ENTER){
      oscSelect = OSCB2;
      oscName = "B2";
      oscOffset = 96;
      menuChange(OSCMENU);
    }
    break;
  case 6:
    if (eventType == ENCODER){
      menuDraw(9,1);
    }
    else if (eventType == ENTER){
      oscSelect = OSCB3;
      oscName = "B3";
      oscOffset = 112;
      menuChange(OSCMENU);
    }
    break;
  case 7:
    if (eventType == ENCODER){
      menuDraw(3,2);
    }
    else if (eventType == ENTER){
      oscName = "MxA";
      oscOffset = 0;
      menuChange(RAW);
    }
    break;
  case 8:
    if (eventType == ENCODER){
      menuDraw(5,2);
    }
    else if (eventType == ENTER){
      oscName = "MxB";
      oscOffset = 64;
      menuChange(RAW);
    }
    break;
  case 9:
    if (eventType == ENCODER){
      menuDraw(7,2);
    }
    else if (eventType == ENTER){
      oscName = "Ctrl";
      oscOffset = 128;
      menuChange(RAW);
    }
    break;
  case 10:
    menuDraw(3,3);//call menu drawing function
    break;
  case 11://Name
    if (eventType == ENCODER){
      menuDraw(8,3);
    }
    else if (eventType == ENTER){
      menuChange(PATCHNAMEMENU);
    }
    break;
  case 12://End
    menuState = 1;//call menu drawing function
    menuDraw(3,0);
    break;
    //
  case 20: //Main Menu Osc A1
    break;
  case 21:
    menuDraw(10,1);//call menu drawing function
    break;
  case 22:
    menuDraw(0,2);//call menu drawing function
    break;
  case 23:
    menuDraw(10,2);//call menu drawing function
    break;

  }
}

void menuOscSwitch(){

  // Draw menu if required
  if (eventType == REFRESH){
    menuCursorSet(0,1);
    //menuRef("[Osc " + oscName + " Menu]", " Amplitude ADSR", " Frequency Waveform", " Raw");
    lcd.clear();
    lcd.print("[Osc " + oscName + " Waveform]");
    lcd.setCursor(0,1);
    lcd.print(F(" Amplitude ADSR"));
    lcd.setCursor(0,2);
    lcd.print(F(" Frequency Waveform"));
    lcd.setCursor(0,3);
    lcd.print(F(" Raw"));

    menuDraw(0,1);
    eventType = NONE;
  }
  if (eventType == EXIT){
    menuChange(MAINMENU);
  }

  switch(menuState){
  case 0:
    menuState = 5;
    menuDraw(0,3);//call menu drawing function
    break;
  case 1:
    if (eventType == ENCODER){
      menuDraw(0,1);//call menu drawing function
    }
    else if (eventType == ENTER){
    }
    break;
  case 2:// ADSR
    if (eventType == ENCODER){
      menuDraw(10,1);//call menu drawing function
    }
    else if (eventType == ENTER){
      menuChange(ADSR);
    }
    break;
  case 3:
    if (eventType == ENCODER){
      menuDraw(0,2);//call menu drawing function
    }
    else if (eventType == ENTER){
    }
    break;
  case 4:
    if (eventType == ENCODER){
      menuDraw(10,2);//call menu drawing function
    }
    else if (eventType == ENTER){
      menuChange(OSCWAVEMENU);
    }
    break;
  case 5:// Raw Edit mode *Development Only*
    if (eventType == ENCODER){
      menuDraw(0,3);//call menu drawing function
    }
    else if (eventType == ENTER){
      menuChange(RAW);
    }
    break;
  case 6:
    menuState = 1;
    menuDraw(0,1);//call menu drawing function
    break;
  }
}




void menuOscWaveform(){
  // Draw menu if required
  if (eventType == REFRESH){
    oscStatus = patch[oscSelect];
    menuCursorSet(0,1);
    //menuRef("[Osc " + oscName + " Waveform]", " Wave=", " On  Abs  Ovf  Bnd", " PMd  PWM=");
    lcd.clear();
    lcd.print("[Osc " + oscName + " Waveform]");
    lcd.setCursor(0,1);
    lcd.print(F(" Wave="));
    lcd.setCursor(0,2);
    lcd.print(F(" On  Abs  Ovf  Bnd"));
    lcd.setCursor(0,3);
    lcd.print(F(" PMd  PWM="));

    menuDraw(0,1);  
    //Update Waveform Type
    waveTypePrint (patch[oscSelect] & 0x07);
    //Update Abs
    printStar(8,2, (oscStatus & 0x08));
    //Update Ovf
    printStar(13,2, (oscStatus & 0x10));
    //Update Bnd
    printStar(18,2, (oscStatus & 0x20));
    //Update On
    printStar(3,2, (oscStatus & 0x80));
    //
    eventType = NONE;
  }
  if (eventType == EXIT){
    menuChange(OSCMENU);
  }

  switch(menuState){
  case 0:
    menuState = 7;
    menuDraw(5,3);
    break;
  case 1: //Wave
    if (eventType == ENCODER){
      menuDraw(0,1);
    }
    else if (eventType == ENTER){
      menuChange(4); //waveformSelect    
    }
    break;
  case 2: //On
    if (eventType == ENCODER){
      menuDraw(0,2);
    }
    else if (eventType == ENTER){
      oscStatus = (oscStatus ^ 0x80); //Bitwize XOR flips bit 8
      writeOneMask(oscSelect, 0x80, oscStatus);
      printStar(3,2, (oscStatus & 0x80));
    }
    break;
  case 3: //Abs
    if (eventType == ENCODER){
      menuDraw(4,2);
    }
    else if (eventType == ENTER){
      oscStatus = (oscStatus ^ 0x08); //Bitwize XOR flips bit 4
      writeOneMask(oscSelect, 0x08, oscStatus);
      printStar(8,2, (oscStatus & 0x08));
    }
    break;
  case 4: //Ovf
    if (eventType == ENCODER){
      menuDraw(9,2);
    }
    else if (eventType == ENTER){
      oscStatus = (oscStatus ^ 0x10); //Bitwize XOR flips bit 5
      writeOneMask(oscSelect, 0x10, oscStatus);
      printStar(13,2, (oscStatus & 0x10));
    }
    break;
  case 5: //Bnd
    if (eventType == ENCODER){
      menuDraw(14,2);
    }
    else if (eventType == ENTER){
      oscStatus = (oscStatus ^ 0x20); //Bitwize XOR flips bit 6
      writeOneMask(oscSelect, 0x20, oscStatus);
      printStar(18,2, (oscStatus & 0x20));
    }
    break;
  case 6: //PMd
    if (eventType == ENCODER){
      menuDraw(0,3);
    }
    else if (eventType == ENTER){
    }
    break;
  case 7: //PWM
    if (eventType == ENCODER){
      menuDraw(5,3);
    }
    else if (eventType == ENTER){
    }
    break;
  case 8:
    menuState = 1;
    menuDraw(0,1);
    break;
  }
}// End menuOscWaveform

void waveformSelect(){
  if (eventType == REFRESH){

    lcd.setCursor(0,1);
    lcd.print(":");//test
    menuState = waveform;
    oldWaveform = waveform;
    eventType = NONE;
  }
  if (eventType == EXIT){
    writeOneMask(oscSelect, 0x07, oldWaveform);
    waveform = oldWaveform;
    menuChange(OSCWAVEMENU);
  }
  if (menuState < 0){
    menuState = 7;
  }
  if (menuState > 7){
    menuState = 0;
  }

  //Select Waveform
  if (eventType == ENCODER){
    waveTypePrint(menuState);
    writeOneMask(oscSelect, 0x07, menuState);     
  }
  else if (eventType == ENTER){
    menuChange(OSCWAVEMENU);
  } 
}

void menuEnterSwitch(){
}

void menuExitSwitch(){
}




void savePatch(){
  if (eventType == REFRESH){
    menuCursorSet(0,0);
    menuRef("", "    Save Patch?", "", "");
    eventType = NONE;
  }
  else if (eventType == ENTER){
    patchWrite(patchNumber);
    menuChange(PATCHMENU);
  }
  else if (eventType == EXIT){
    menuChange(PATCHMENU);
  }  
}

void patchNameSet(){
  if (eventType == REFRESH){
    menuCursorSet(1,2);
    menuRef("[Edit Name]", "", "", "");
    lcd.setCursor(0,1);
    lcd.print("[");
    lcd.print(patchName);
    lcd.print("]");
    printCursor (1, 2, 1);
    eventType = NONE;
    editMode = 0;
  }
  else if (eventType == ENCODER){
    if (editMode == 0){
      if (menuState < 1) menuState = 16;
      if (menuState > 16) menuState = 1;
      printCursor(menuState,2,1);
    }
    if (editMode == 1){
      if (encoderDirection == true){ //CW Direction
        menuState -= 1; //keep menuState from changing
        patchName[menuState -1] ++;
        if (patchName[menuState -1] > 126) patchName[menuState -1] = 32;
        lcd.setCursor(menuState,1);
        lcd.print(patchName[menuState -1]);
      }
      else if (encoderDirection == false){ //CCW Direction
        menuState += 1; //keep menuState from changing
        patchName[menuState -1] --;
        if (patchName[menuState -1] < 32) patchName[menuState -1] = 126;
        lcd.setCursor(menuState,1);
        lcd.print(patchName[menuState -1]);
        //lcd.print(menuState);//test
      }
    }
  }
  else if (eventType == ENTER){
    editMode = 1 - editMode; //Flips 1 to 0 and visa-versa
    printCursor(menuState,2,editMode + 1);
  }
  else if (eventType == EXIT){
    menuChange(MAINMENU);
  }  
}//patchNameSet End

// ADSR Menu Function
void adsr(){
  if (eventType == REFRESH){
    menuCursorSet(0,1);
    lcd.clear();
    lcd.print("[Osc " + oscName + " ADSR]");
    lcd.setCursor(0,1);
    lcd.print(F(" Attack =      Lv=  "));
    lcd.setCursor(0,2);
    lcd.print(F(" Decay  =      Lv=  "));
    lcd.setCursor(0,3);
    lcd.print(F(" Release=      Lv=  "));
    adsrSet(0,1,13,false,0);
    adsrSet(14,1,13,true,0);
    adsrSet(0,2,14,false,1);
    adsrSet(14,2,14,true,0);
    adsrSet(0,3,15,false,1);
    adsrSet(14,3,15,true,0);
    menuDraw(0,1);
    eventType = NONE;
    editMode = 0;
  }
  if (eventType == EXIT){
    menuChange(MAINMENU);
  }

  if (eventType == ENTER) editMode = 1 - editMode;
  if (editMode == 0) {
    backState = menuState;
    printCursor(cursorX, cursorY, 3);
  }
  if (editMode == 1) {
    menuState = backState;
    printCursor(cursorX, cursorY, 4);
  }

 if (menuState < 1) menuState = 6;
 if (menuState > 6) menuState = 1;


  switch(menuState){
    
  case 1: 
    if (eventType == ENCODER){
      adsrSet(0,1,13,false,0);
    }
    break;
  case 2: 
    if (eventType == ENCODER){
      adsrSet(14,1,13,true,0);
    }
    else if (eventType == ENTER){    
    }
    break;
  case 3: 
    if (eventType == ENCODER){
      adsrSet(0,2,14,false,1);
    }
    else if (eventType == ENTER){  
    }
    break;
  case 4: 
    if (eventType == ENCODER){
      adsrSet(14,2,14,true,0);
    }
    else if (eventType == ENTER){   
    }
    break;
  case 5: 
    if (eventType == ENCODER){
      adsrSet(0,3,15,false,1);
    }
    else if (eventType == ENTER){    
    }
    break;
  case 6: 
    if (eventType == ENCODER){
      adsrSet(14,3,15,true,0);
    }
    else if (eventType == ENTER){   
    }
    break;
  }
}//adsr End

/* adsrSet - sets adsr registers on soundgin
 parameters - x,y = coordinates of cursor
 param = memory location of sound gin resister
 byteHigh = The soundgin uses the high nibble of the byte for one parameter
 and the low nibble for the other. This arg lets the function know
 which part of the byte to change
 offset = There is a global int array that stores the time values in milliseconds.
 This is only for the menu display to display the times correctly
 */
void adsrSet(byte x, byte y, byte param, boolean byteHigh, byte offset){

  if (eventType == REFRESH){
    if (byteHigh) {
      lcd.setCursor(x + 4,y);
      lcd.print(patch[param + oscOffset] >> 4);
    }
    else {
      lcd.setCursor(x + 9,y);
      lcd.print(pgm_read_word(&adsrVal[(patch[param + oscOffset] & 0x0F) * 2 + offset]));
    }
    return;
  }

  if (editMode == 0) menuDraw(x,y);

  else if (editMode == 1){
    if (encoderDirection == true){
      if (byteHigh) patch[param + oscOffset] += 16;
      else {
        if ((patch[param + oscOffset] & 0x0F) != 0x0F)
          patch[param + oscOffset] += 1;
      }
      writeOneByte(param + oscOffset, patch[param + oscOffset]);
    }
    else if (encoderDirection == false){
      if (byteHigh) patch[param + oscOffset] -= 16;
      else {
        if ((patch[param + oscOffset] & 0x0F) != 0x00)
          patch[param + oscOffset] -= 1;
      }
      writeOneByte(param + oscOffset, patch[param + oscOffset]);
    }


    if (byteHigh) {
      lcd.setCursor(x + 4,y);
      lcd.print("  ");
      lcd.setCursor(x + 4,y);
      lcd.print(patch[param + oscOffset] >> 4);
    }
    else {
      lcd.setCursor(x + 9,y);
      lcd.print("     ");
      lcd.setCursor(x + 9,y);
      lcd.print(pgm_read_word(&adsrVal[(patch[param + oscOffset] & 0x0F) * 2 + offset]));
    }
  }
}


## NaV1_V1_master
/**********************************************************************
 *                 NaV-1 Arduino Soundgin Synth V1.0                  *
 *               Visit - notesandvolts.com for tutorial               *
 *                                                                    *
 *               Requires Arduino Version 1.0 or Later                *
 **********************************************************************/

#include <SoftwareSerial.h>
#include <MIDI.h>
#include <LiquidCrystal.h>
#include <Wire.h>
#include <avr/pgmspace.h>

#define CTS 2 
#define txPin 3
#define rxPin 4
#define ROTARY_PIN1 5
#define ROTARY_PIN2 7
#define LCD_RS 6
#define LCD_EN 8
#define LCD_DB4 A0
#define LCD_DB5 A1
#define LCD_DB6 A2
#define LCD_DB7 A3
#define LED 13
#define BUTTON_ENTER 9
#define BUTTON_EXIT 10

#define NumNotes 96

#define TIMEOUT 300
#define DIR_CCW 0x10
#define DIR_CW 0x20

#define ENCODER 1
#define ENTER 2
#define EXIT 3
#define REFRESH 4
#define NONE 5

#define PATCHMENU 0
#define MAINMENU 1
#define OSCMENU 2
#define OSCWAVEMENU 3
#define PATCHNAMEMENU 5
#define PATCHSAVE 99
#define RAW 6
#define ADSR 7
#define EEPROM1 0x50 //Address of eeprom chip

#define OSCA1 0x10
#define OSCA2 0x20
#define OSCA3 0x30
#define OSCB1 0x50
#define OSCB2 0x60
#define OSCB3 0x70


// Create a SoftwareSerial object named "Soundgin"
SoftwareSerial soundgin = SoftwareSerial(rxPin, txPin);

// Initialize the LCD and Assign pins
LiquidCrystal lcd(LCD_RS, LCD_EN, LCD_DB4, LCD_DB5, LCD_DB6, LCD_DB7);

//TEST ONLY
// variables created by the build process when compiling the sketch
extern int __bss_end;//Memory test
extern void *__brkval;//Memory test
// Global Variables
byte currentNote = 0;
byte newNote = 0;
volatile unsigned char state = 0;
boolean enterPress = false;
boolean exitPress = false;
unsigned long enterTime;
unsigned long exitTime;
int myCursor = 10;

int menuState = 1;
int menuLevelState = 0; //Level of Menu
//boolean menuRefresh = true;
int cursorX = 0;
int cursorY = 0;
byte editMode = 0; //Used by name func to edit letters

int backState = 1;
byte eventType = 0;
boolean encoderDirection; // True for CW. False for CCW
byte oscSelect = 0; // Stores selected Oscillator
String oscName;
byte oscOffset = 0; //Used by RAW EDIT
byte oscStatus = 0; // Stores Osc status byte
byte waveform = 0; // Stores wave type
byte oldWaveform = 0; // old wave for restore

// Array of Patch parameters
byte patchNumber = 0;
byte patch[128] = {//Holds 128 Byte Soundgin Parameters
  7,0,0,0,0,0,0,0,127,0,0,0,0,0,0,0, //Mix A
  146,0,0,0,0,0,0,0,0,0,0,0,0,240,204,9, //Osc A1
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc A2
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc A3
  7,0,0,0,0,0,0,0,127,0,0,0,0,0,0,0, //Mix B
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc B1
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc B2
  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, //Osc B3
};
char patchName[16] = "Notes and Volts";//Holds patch name

// Lookup Table for Rotary Encoder function
const unsigned char relookup[7][4] PROGMEM = {
  {
    0x0, 0x2, 0x4,  0x0    }
  , 
  {
    0x3, 0x0, 0x1, 0x10    }
  ,
  {
    0x3, 0x2, 0x0,  0x0    }
  , 
  {
    0x3, 0x2, 0x1,  0x0    }
  ,
  {
    0x6, 0x0, 0x4,  0x0    }
  , 
  {
    0x6, 0x5, 0x0, 0x20    }
  ,
  {
    0x6, 0x5, 0x4,  0x0    }
  ,
};

// Lookup Table - Coverts incoming Midi Note Number to Soundgin note number
const byte lookup[NumNotes] PROGMEM = {
  0,1,2,3,4,5,6,7,8,9,10,11,
  16,17,18,19,20,21,22,23,24,25,26,27,
  32,33,34,35,36,37,38,39,40,41,42,43,
  48,49,50,51,52,53,54,55,56,57,58,59,
  64,65,66,67,68,69,70,71,72,73,74,75,
  80,81,82,83,84,85,86,87,88,89,90,91,
  96,97,98,99,100,101,102,103,104,105,106,107,
  112,113,114,115,116,117,118,119,120,121,122,123
};

// Soundgin Initialization Commands
// **27 = Command Charater (Precedes all commands)**
// **27,1,Memory,Param (Stores Parameter in Memory Location)**
byte sgInit[] PROGMEM = {
  27,6, // (6 = Clear Mixers A & B)
  27,84, 27,85, 27,86, // (84,85,86 = Release Osc A1,A2,A3)
  27,116, 27,117, 27,118,  // (116,117,118 = Release Osc B1,B2,B3)
  27,1,136,127, // (136 = Set Master Volume to 127 - FULL)
  27,1,8,127, // (8 = Set Mixer A Volume to 127 - FULL)
  27,1,72,127, // (72 = Set Mixer B Volume to 127 - FULL)
  27,1,0,1, // Send Osc A1 to Mixer A
  27,1,1,0, // Turn Off PWM for Osc A1
  27,1,16,146, // 16 = Osc A Control = OvF Mode on, Saw Wave set
  27,1,29,240 ,27,1,30,204, 27,1,31,9 };// 29,30,31 = Osc A1 Atack,Decay,Release


//Lookup Table ADSR Values
const unsigned int adsrVal[] PROGMEM = {
  2,6,8,24,16,48,24,72,38,114,56,168,68,204,80,240,100,300,
  250,750,500,1500,800,2400,1000,3000,2800,8400,5600,16800,11200,33600
};

void setup()
{
  //Set Arduino Pins
  pinMode (LED, OUTPUT);
  pinMode (rxPin, INPUT);
  pinMode (txPin, OUTPUT);
  pinMode (CTS, INPUT);

  pinMode(ROTARY_PIN1, INPUT); //Set Rotary Encoder Pin 1 to Input
  digitalWrite(ROTARY_PIN1, HIGH); //Enable on board resistor
  pinMode(ROTARY_PIN2, INPUT);
  digitalWrite(ROTARY_PIN2, HIGH);

  pinMode(BUTTON_ENTER,INPUT);            // default mode is INPUT
  digitalWrite(BUTTON_ENTER, HIGH);     // Turn on the internal pull-up resistor
  pinMode(BUTTON_EXIT,INPUT);            // default mode is INPUT
  digitalWrite(BUTTON_EXIT, HIGH);     // Turn on the internal pull-up resistor

  Wire.begin();
  soundgin.begin(9600); //Set Software Serial Baud Rate to 9600
  //initPatch(); //Run patch function - Initializes and sets up sound on Soundgin Chip
  MIDI.begin(MIDI_CHANNEL_OMNI); // Initialize the Midi Library.
  MIDI.setHandleNoteOn(MyHandleNoteOn); //set MyHandleNoteOn as the callback function

    // Print to LCD
  lcd.begin(20, 4);
  delay(500);
  lcd.print(F(" NOTESANDVOLTS.COM "));
  lcd.print(F("         1.0 "));
  eepromInit();
  initPatch(); //Run patch function - Initializes and sets up sound on Soundgin Chip
  menuChange(PATCHMENU);
  eventType = REFRESH;
}

void loop() // Main Loop
{
  MIDI.read(); // Continually check RX for incoming Midi Commands
  menu(); // Check state of Rotary Encoder and Buttons - Update LCD
}





## Eeprom info and Code

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


