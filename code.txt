 /*
  Made by V.V.Poliakov (ParrKenna) for WWSERVICE. 

  Comments are for following code, that goes after it:

  //COMMENT 
      |
      V
  code();
  
  Some questionable moments and my contacts are listed and explained in the end of this code/document.
*/

//Libraries.

#include <PZEM004Tv30.h>

#include <LiquidCrystal.h>

#include <Wire.h>

#include <EEPROM.h>

/*
 Buttons ID's for analog keyboard.
 Same codes means the same button 
 used for different functions. 
*/
#define STOP_BUTTON 0
#define AUTO_BUTTON 1
#define PLUS_BUTTON 1
#define MANUAL_BUTTON 2
#define MINUS_BUTTON 2
#define EXIT_BUTTON 3
#define SET_BUTTON 4

//Buttons on the shield. 

#define BUTTON_RIGHT  0
#define BUTTON_UP     1
#define BUTTON_DOWN   2
#define BUTTON_LEFT   3
#define BUTTON_SELECT 4
#define ANYBUTTON 5

//Pins on shield.

#define RELAYPIN 2 
#define SUPPLYCONTROLPIN 3
#define KEYBOARDPIN A0

//Work modes.

#define STOPMODE 1
#define AUTOMODE 2
#define MANUALMODE 3

/* 
 DEFAULT values of settings. 
 Used ONLY for debug, 
 setting to default and 
 setting up new device.
*/

#define amperageMAXDEFAULT 7.95
#define amperageMINDEFAULT 3.95
#define voltageMAXDEFAULT 245
#define voltageMINDEFAULT 195
#define wattageMultiplierDEFAULT 0.9

//Error codes.

#define err_not_stopped 1
#define err_amperage_high 2
#define err_amperage_low 3
#define err_voltage_low 4
#define err_voltage_high 5

//Custom symbols for the LCD

byte lockerSymbol[8]{
  0b00100,
  0b01010,
  0b01010,
  0b01010,
  0b11111,
  0b11011,
  0b11011,
  0b01110,
};

byte turningOnModeSymbol[8]{
  0b00000,
  0b00000,
  0b00100,
  0b10101,
  0b10101,
  0b10001,
  0b01110,
  0b00000,
};

//Inititalizing of LCD and PZEM-004t

LiquidCrystal lcd(8, 9, 4, 5, 6, 7);

PZEM004Tv30 pzem(Serial);

//Comparizing with current signal of Supply control pin.

unsigned short int supplyTurningOn = 1; 

//Locks&clocks for starting and stopping
bool isLockedForTurningOn;
bool isLockedForShuttingDown;
int lockTimerTurningOnINT;
int lockTimerShuttingDownINT;
uint32_t turningOnLockTimer;
uint32_t shuttingDownLockTimer;

/*
 Function used when error occured.
 Locking turning on, for the milliseconds that is defined by setTimer variable.
*/

void lockTurningOn(int setTimer){
    //Lock.
    isLockedForTurningOn = true;
    //Count current time.
    turningOnLockTimer = millis();
    //Setting or updating current delay.
    lockTimerTurningOnINT = setTimer;
}

/*
 Function used when the AUTO or MANUAL mode is only started.
 Locking turning on, for the milliseconds that is defined by setTimer variable.
 Locking shutting down by the error catcher.
 User can shut down device manually, it can't be locked.
*/

void lockShuttingDown(int setTimer){
  //Locking.
  isLockedForShuttingDown = true;
  //Count current time.
  shuttingDownLockTimer = millis();
  //Setting or updating current delay.
  lockTimerShuttingDownINT = setTimer; 
}

/*Function that is used to get the current state of concrete or any button.
Transforms analog signal of keyboard to digital state (0 or 1) of buttons.

1 is not touched.
0 is touched.

 */

bool getButtonState(int button){
  //Check for button ID. 
  if(button == ANYBUTTON){
    /*
     If button ID is 5(Any button is needed to be touched), 
     check the current keyboard signal. If ANY button touched,
     it could be less than 1000.*/
    if(analogRead(KEYBOARDPIN) > 1000) {return 1;
    }else{
      return 0;
      }

  } else{
    //If button id is concrete, creating variable that will save ID of tuched button.
    int realpressedbutton;
    //Getting signal of keyboard.
    int keyboardsignal = analogRead(KEYBOARDPIN);
    //Comparising the signal with buttons signal area map. The current is for D1ROBOT
    if(keyboardsignal < 50) {
      realpressedbutton = BUTTON_RIGHT;
    } else if(keyboardsignal < 250) {
      realpressedbutton = BUTTON_UP;
    } else if(keyboardsignal < 450) {
      realpressedbutton = BUTTON_DOWN;
    } else if(keyboardsignal < 650) {
      realpressedbutton = BUTTON_LEFT;
    } else if(keyboardsignal < 850) {
      realpressedbutton = BUTTON_SELECT;
    } else {
      return 1;
    }
    //Check for the according currently pressed button and button, that is checking for be touched.
    if(button == realpressedbutton){
      return 0;
    } else { 
      return 1;}
  }
}

//Variables for saving the current and previous(postfix "prev") values got from PZEM-004t

int voltageINTERIOR = 000;
float amperageINTERIOR = 0.00;
float wattageINTERIOR = 0.00;
int voltageINTERIORprev = 000;
float amperageINTERIORprev = 0.00;
float wattageINTERIORprev = 0.00;

//Variable to save the currently active work mode. Work modes IDs defined at line 54.

short int menuWorkMode = STOPMODE;

//Function that saving current mode to EEPROM. Saves only STOP or AUTO modes.

void writeCurrentToEEPROM(bool toWrite){
  EEPROM.put(50,toWrite);
}

//Getting mode to start from EEPROM.
bool checkForCurrentSetupMODE(){
  bool whattoreturn;
  EEPROM.get(50,whattoreturn);
  return whattoreturn;
}

// Variables to control the settings menu. 0 NO 1 YES.

// Settings menu status.
bool isSettingsMenuActive = false; 
// Variable used to save current settings menu page opened.
int settingmap; 
// Variable used to keep opened or closed concrete setting page. 
bool isConcreteSettingChoosed = false;
/* Variable used analogically to settingmap, 
but only for settings that have concrete page.*/
unsigned int choosedConcreteSettingID;

// Current settings. Used to catch errors.

float amperageMAX;
float amperageMIN;
int voltageMAX;
int voltageMIN;
float wattageMultiplier;

//Temporary values for settings, to change them.

float floatTEMPORARYSetting;
int intTEMPORARYSetting;

//Variables used in error catch system.

uint32_t errorOutputDisableTimer;
bool isErrorActive = false; 
unsigned int savedErrorCode;
int savedMsToDisable;

//Variable for screen updater. Used to check if update screen is needed.

bool isNeededUpdate = 1;

//Screen updater. Checks if needed to clear it, and does it if yes. 
void updateIfNeeded(){
  if(isNeededUpdate){
    lcd.clear();
    isNeededUpdate = false;
  }
}

//Exit from setting menu and clearing after. 

bool isAUTOactive = false;

void settingsMenuExit(){
  isNeededUpdate = true;
  isSettingsMenuActive=0;
  menuWorkMode = STOPMODE;
      isAUTOactive=false;


}

//Getting settings from EEPROM to RAM. 

void getSettingsFromEEPROM(){
  EEPROM.get(0,amperageMAX);
  EEPROM.get(5,amperageMIN);
  EEPROM.get(10,voltageMAX);
  EEPROM.get(15,voltageMIN);
  EEPROM.get(20,wattageMultiplier);
}

//Setting to default. Default setting on line 60. Accompanied with visual effects.

void setDefaultSettingsToEEPROM(){
  isNeededUpdate=true;
  updateIfNeeded();
  lcd.setCursor(0, 0);
  lcd.print("SETTING TO");
  lcd.setCursor(0, 1);
  lcd.print("DEFAULT");
  EEPROM.put(0,amperageMAXDEFAULT);
  EEPROM.put(5,amperageMINDEFAULT);
  EEPROM.put(10,voltageMAXDEFAULT);
  EEPROM.put(15,voltageMINDEFAULT);
  EEPROM.put(20,wattageMultiplierDEFAULT);
  delay(100);
  lcd.print(" .");
  delay(250);
  lcd.print(" .");
  delay(250);
  lcd.print(" .");
  delay(500);
  isNeededUpdate=true;
  updateIfNeeded();
  lcd.setCursor(0, 0);
  lcd.print("DONE");
  delay(3000); 
}

//Visual part and activity setter of Error catching module. Compares Error codes and updates screen.

void errorOnlyOutput(unsigned int errorCode, int msToDisable){
  savedErrorCode=errorCode;
  savedMsToDisable=msToDisable;
  //Activity setter.
  if(isErrorActive==false){
    errorOutputDisableTimer = millis();
    isErrorActive=true;
  }
  updateIfNeeded();
  lcd.setCursor(0,0);
  //Following part is used in every error except "not stopped", on top line of LCD.
  if(errorCode!=1){
    lcd.print("ERROR");
  }
  //Prints current error description on bottom line of LCD, except first.
  lcd.setCursor(0,1);
  switch(errorCode){
    case err_not_stopped:{
      //Prints "NOT STOPPED" on top line of LCD.
      lcd.clear();
      lcd.setCursor(0,0);
      lcd.print("NOT STOPPED");
      break;}
      //Any other prints "ERROR" in top line and current error on bottom line.
    case err_amperage_high:{
      lcd.print("AMPERAGE HIGH");
      break;
    }
    case err_amperage_low:{
      lcd.print("AMPERAGE LOW");
      break;
    }
    case err_voltage_low:{
      lcd.print("VOLTAGE LOW");
      break;
    }
    case err_voltage_high:{
      lcd.print("VOLTAGE HIGH");
      break;
    }
  }
}

//The home page visual

void menuOutput(){

  //ONLY FOR TEST
  //The test used for check keyboard.

 /* if(getButtonState(STOP_BUTTON)){
    lcd.setCursor(0,0);
    lcd.print("stop");
  }else if(getButtonState(PLUS_BUTTON)){
    lcd.setCursor(0,0);
    lcd.print("+/auto");
  } else if(getButtonState(MINUS_BUTTON)){
    lcd.setCursor(0,0);
    lcd.print("-/man");
  }else if(getButtonState(EXIT_BUTTON)){
    lcd.setCursor(0,0);
    lcd.print("EXT");
  }else if(getButtonState(SET_BUTTON)){
    lcd.setCursor(0,0);
    lcd.print("set");
  }*/

  //ONLY FOR TEST

  //Checking current parameters, comparizing with previous. If found diffent, clears screen.
  if(voltageINTERIOR!=voltageINTERIORprev){
    isNeededUpdate=true;
  }
  else if(amperageINTERIOR!=amperageINTERIORprev){
    isNeededUpdate=true;
  }
  else if(wattageINTERIOR!=wattageINTERIORprev){
    isNeededUpdate=true;
  }
  updateIfNeeded();

  //Printing information to screen. It is a home page of device, used most time.

  lcd.setCursor(0,0);
  lcd.print("U:");
  lcd.print(voltageINTERIOR);
  lcd.print("V");
  lcd.setCursor(8, 0);
  lcd.print("I:");
  lcd.print(amperageINTERIOR,2);
  lcd.print("A");
  lcd.setCursor(0,1);
  lcd.print("P:");
  lcd.print(wattageINTERIOR,2);
  lcd.print("kW");
  lcd.setCursor(10,1);

  //Printing custom symbols to show the turning on and shutting down locks.

  if(isLockedForTurningOn==true){
    lcd.print(char(1));
  }else if(isLockedForShuttingDown==true){
    lcd.print(char(2));
  } else{
     lcd.print(" ");
  }
  
  //Printing current work mode on screen.

  lcd.setCursor(12, 1);
  switch(menuWorkMode){
    case STOPMODE:{ if(isAUTOactive==false){
      lcd.print("STOP"); 
    } else {
      lcd.print("AUTO");
    }
    break;}
    case AUTOMODE:{ lcd.print("AUTO");
    break; 
    }
    case MANUALMODE:{ lcd.print("MAN."); 
    break;}
  }
  
  //Saving current parameters for next comparison.
  voltageINTERIORprev=voltageINTERIOR;
  amperageINTERIORprev=amperageINTERIOR;
  wattageINTERIORprev=wattageINTERIOR;
}

//Variable to save mode which will be saved to and loaded from EEPROM, and preventors for correct buttons work.

bool afterlockingmode;
bool istouchedSET_BUTTONPreventor=false;
bool istouchedEXIT_BUTTONPreventor=false;

//Function for settings menu, first level - user can choose which setting will be changed.

void settingsMapFunc(String print1, String print2, unsigned short int up, unsigned short int down, unsigned short int CCSID, unsigned short int setChoise, unsigned short int extdestiantionfromlayer2, unsigned short int setlayer2){
  
  //Printing current setting in list

  updateIfNeeded();
  lcd.setCursor(0, 0);
  lcd.print(print1);
  lcd.setCursor(0, 1);
  lcd.print(print2);

  //Ensures that user unpressed buttons before next moves.

  if(getButtonState(SET_BUTTON)==1){
    istouchedSET_BUTTONPreventor=false;
  }
  if(getButtonState(EXIT_BUTTON)==1){
    istouchedEXIT_BUTTONPreventor=false;
  }
  //delay(250);

  //Menu navigation, to next and previous setting to choose.

  if(getButtonState(PLUS_BUTTON)==0){
    isNeededUpdate=true;
    settingmap=up;
    //delay(250);
  }else if(getButtonState(MINUS_BUTTON)==0){
    isNeededUpdate=true;
    settingmap=down;
    //delay(250);

    // Menu navigation to exit from setting to main screen.
  }else if(getButtonState(EXIT_BUTTON)==0&&istouchedEXIT_BUTTONPreventor==false){
    istouchedEXIT_BUTTONPreventor=true;
    if(extdestiantionfromlayer2==0){
      isNeededUpdate=true;
      settingsMenuExit();
      //delay(250);

      // Also, for simulation of second layer.
    }else if(extdestiantionfromlayer2!=0){
      isNeededUpdate=true;
      settingmap=extdestiantionfromlayer2;
      //delay(250);
    }
    //delay(250);
    
    //Menu navigation to choose concrete setting to change.
  } else if(getButtonState(SET_BUTTON)==0&&istouchedSET_BUTTONPreventor ==false){
    istouchedSET_BUTTONPreventor=true;
    istouchedEXIT_BUTTONPreventor=false;

    //If 0, do nothing. Used in case of "SETTINGS" welcome page.

    if(setChoise==0){
      isNeededUpdate=true;

    //If 1, user choised concrete setting.

    }else if(setChoise==1){ 
      istouchedSET_BUTTONPreventor = true;
      choosedConcreteSettingID=CCSID;
      isNeededUpdate=true;
      isConcreteSettingChoosed=true;

      //In this case, used uneffective method, but more safe than using unspecified pointers.
      //Saving user choice. 

      switch(CCSID){
        case 0:{
          floatTEMPORARYSetting=amperageMAX;
          break;
        }
        case 1:{
          floatTEMPORARYSetting=amperageMIN;
          break;
        }
        case 2:{
          intTEMPORARYSetting=voltageMAX;
          break;
        }
        case 3:{
          intTEMPORARYSetting=voltageMIN;
          break;
        }
        case 4:{
          floatTEMPORARYSetting=wattageMultiplier;
          break;
        }
      }
      isNeededUpdate=true;
      
      //If 2, used simulation of second layer.

    }else if(setChoise==2){
      isNeededUpdate=true;
      settingmap=setlayer2;
      //delay(500);

      //Case 3 used only for setting device to default, and first usage. 

    } else if(setChoise==3){
      setDefaultSettingsToEEPROM();
      settingmap=1;
    }
    //delay(500);
  }
}

//Function for concrete setting change menu, of parameter that is saved as integer. 

void concreteSettingsMapFuncInt(int changescale, String printUpperRow, String measurementUnits, int upperBound, int lowerBound, int EEPROMADRESS){

  //Print concrete setting information. 
  updateIfNeeded();
  lcd.setCursor(0, 0);
  lcd.print(printUpperRow);
  lcd.setCursor(0, 1);
  lcd.print(intTEMPORARYSetting);
  lcd.print(measurementUnits);

  //Ensures that user unpressed buttons before next moves.

  if(getButtonState(SET_BUTTON)==1){
    istouchedSET_BUTTONPreventor=false;
  }
  if(getButtonState(EXIT_BUTTON)==1){
    istouchedEXIT_BUTTONPreventor=false;
  }
  //delay(1000);

  //Change settings, ensure the possibility of changes, and warns user if it can't be changed. Print is at start of function. 

  if(getButtonState(PLUS_BUTTON)==0){
     if(intTEMPORARYSetting<upperBound){
       intTEMPORARYSetting+=changescale;
     }else{
       intTEMPORARYSetting=upperBound;
       lcd.setCursor(15,1);
       lcd.print("!");
     } 
     isNeededUpdate=true;
     //delay(250);
  }else if(getButtonState(MINUS_BUTTON)==0){
    if(intTEMPORARYSetting>lowerBound){
       intTEMPORARYSetting-=changescale;
     }else{
       intTEMPORARYSetting=lowerBound;
       lcd.setCursor(15,1);
       lcd.print("!");
     }
     isNeededUpdate=true;
     //delay(250);

  // Exit from concrete setting menu without saving changes.

  }else if(getButtonState(EXIT_BUTTON)==0&&istouchedEXIT_BUTTONPreventor==false){
    istouchedEXIT_BUTTONPreventor=true;
    isConcreteSettingChoosed=false;
    isNeededUpdate=true;
    //delay(250);

  //Saving changes, without exiting menu. Accompanied with visual effects.

  } else if(getButtonState(SET_BUTTON)==0&&istouchedSET_BUTTONPreventor==false){
    istouchedSET_BUTTONPreventor=true;
    istouchedEXIT_BUTTONPreventor=false;
    isNeededUpdate=true;
    updateIfNeeded();
    lcd.setCursor(0,0);
    lcd.print("SAVING");
    EEPROM.put(EEPROMADRESS,intTEMPORARYSetting);
    delay(250);
    lcd.print(" .");
    delay(250);
    lcd.print(" .");
    delay(250);
    lcd.print(" .");
    delay(250);
    isNeededUpdate=true;
    updateIfNeeded();
    lcd.setCursor(0,0);
    lcd.print("DONE");
    delay(3000);
    isNeededUpdate=true;
  }
}

//Same fuction, but with float. Used uneffective method, explaining in the bottom of code.

void concreteSettingsMapFuncFloat(float changescale, String printUpperRow, String measurementUnits, float upperBound, float lowerBound, int EEPROMADRESS){
  updateIfNeeded();
  lcd.setCursor(0, 0);
  lcd.print(printUpperRow);
  lcd.setCursor(0, 1);
  lcd.print(floatTEMPORARYSetting);
  lcd.print(measurementUnits);
  //delay(1000);
  if(getButtonState(SET_BUTTON)==1){
    istouchedSET_BUTTONPreventor=false;
  }
  if(getButtonState(EXIT_BUTTON)==1){
    istouchedEXIT_BUTTONPreventor=false;
  }

  if(getButtonState(PLUS_BUTTON)==0){
     if(floatTEMPORARYSetting<upperBound){
       floatTEMPORARYSetting+=changescale;
     }else{
       floatTEMPORARYSetting=upperBound;
       lcd.setCursor(15,1);
       lcd.print("!");
     } 
     isNeededUpdate=true;
     //delay(250);
  }else if(getButtonState(MINUS_BUTTON)==0){
    if(floatTEMPORARYSetting>lowerBound){
       floatTEMPORARYSetting-=changescale;
     }else{
       floatTEMPORARYSetting=lowerBound;
       lcd.setCursor(15,1);
       lcd.print("!");
     }
     isNeededUpdate=true;
     //delay(250);
  }else if(getButtonState(EXIT_BUTTON)==0&&istouchedEXIT_BUTTONPreventor==false){
    istouchedEXIT_BUTTONPreventor=true;
    isConcreteSettingChoosed=false;
    isNeededUpdate=true;
    //delay(250);
  } else if(getButtonState(SET_BUTTON)==0&&istouchedSET_BUTTONPreventor==false){
    //delay(4000);
    istouchedSET_BUTTONPreventor=true;
    istouchedEXIT_BUTTONPreventor=false;
      isNeededUpdate=true;
      updateIfNeeded();
      lcd.setCursor(0,0);
      lcd.print("SAVING");
      EEPROM.put(EEPROMADRESS,floatTEMPORARYSetting);
      delay(250);
      lcd.print(" .");
      delay(250);
      lcd.print(" .");
      delay(250);
      lcd.print(" .");
      delay(250);
      isNeededUpdate=true;
      updateIfNeeded();
      lcd.setCursor(0,0);
      lcd.print("DONE");
      delay(3000);
      isNeededUpdate=true;
    
  }
}

//Preventor that will ensure the user seen the error and it's reason. 

bool isButtonTouchedAfterErrorDisable = false;
void errorStopCatcher(){

  //The preventor will cannot be disabling during little time.

  if(millis()-errorOutputDisableTimer>=savedMsToDisable){
    if(isButtonTouchedAfterErrorDisable == false){
      if(getButtonState(ANYBUTTON) == 0){
        isButtonTouchedAfterErrorDisable = true;
      }

      //Also, ensuring preventor will be closed after. 

    } else {
          isErrorActive=false;
          isNeededUpdate=true;
        }
}
}

//Timer, and preventors. Needed to manage settings menu.
uint32_t settingsMenuTimer;
bool isSettingsMenuTimerSet = false;
bool isSettingsMenuWelcomePageActive = false;
unsigned short int isSetUntouched=0;

//Variable to turn on device after starting up, if needed.

bool neededToTurnOn;

//Setup is called once at start of devices work.
void setup() {

  //Starting up the LCD, and UART for PZEM-004t

  lcd.begin(16, 2);  
  Serial.begin(9600);

  //Turning on device if needed
  if(checkForCurrentSetupMODE()==true){
    afterlockingmode = true;
    neededToTurnOn = true;
    lockShuttingDown(5000);
    isAUTOactive=true;
  } else{
    afterlockingmode = false;
    neededToTurnOn = true;
    isAUTOactive = false;
  }

  //Starting up pins, LCD's cursor, settings of device.

  pinMode(RELAYPIN,OUTPUT); // RELAY
  pinMode(KEYBOARDPIN,INPUT); //KEYBOARD
  pinMode(SUPPLYCONTROLPIN,INPUT); //SUPPLY CONTROL 
  lcd.setCursor(0,0);
  getSettingsFromEEPROM();

  //Loading custom symbols to LCD, end of starting up.
  lcd.createChar(1,lockerSymbol);
  lcd.createChar(2,turningOnModeSymbol);
  lockTurningOn(5000);
  isSettingsMenuTimerSet= false;
}

//Controller to lock shutting down for start of work.

bool turnOnOnlyOnce = true;

//This function is repeating during all devices work time.

void loop() {

// Read and check info from PZEM-004t

  voltageINTERIOR = pzem.voltage();
  amperageINTERIOR = pzem.current();
  wattageINTERIOR = pzem.power()/1000;
  if(isnan(voltageINTERIOR)){
    voltageINTERIOR=0;
  }
  if(isnan(amperageINTERIOR)){
    amperageINTERIOR=0;
  }
  if(isnan(wattageINTERIOR)){
    wattageINTERIOR=0;
  }
  
//Main menu. If stopped or auto turned on, writes it to EEPROM. 

if(menuWorkMode==STOPMODE){
  if(checkForCurrentSetupMODE()==true&&isAUTOactive==false){
        writeCurrentToEEPROM(false);
      }
      } else if(menuWorkMode==AUTOMODE||isAUTOactive==true){
      if(checkForCurrentSetupMODE()==false){
        writeCurrentToEEPROM(true);
      }}

//Specifical requirement from customer:
/* There must be supply control pin
When the auto mode is turned on, it must control the power supply. 
When applying power to pin, it must apply power throw relay, 
if no - relay must block power.*/
/*How it works: almost all logic is checking one of two variables to check activity of auto mode:
The menuWorkMode and isAUTOactive. First variable is control of power supply, monitoring and else 
general functions, second - specifical functions like saving workmode, visual, etc. 

This module controlling the menuWorkMode, but saving AUTO as active.*/

  updateIfNeeded();
  if(isAUTOactive==true){
    if(digitalRead(SUPPLYCONTROLPIN) == supplyTurningOn){
      menuWorkMode = AUTOMODE;
          isLockedForTurningOn = false;
          if( turnOnOnlyOnce==true){
            lockShuttingDown(5000);
            
          }
          turnOnOnlyOnce = false;
    } else if (digitalRead(SUPPLYCONTROLPIN) != supplyTurningOn){
      menuWorkMode = STOPMODE; 
      isLockedForShuttingDown=false;
     isLockedForTurningOn = false;
      turnOnOnlyOnce = true;

    }
  }

  //This prevents next code during errors.  

  if(isErrorActive==true){
    errorStopCatcher();

  //If no errors are active at the moment, the next code will control work modes. 
  /*How it works:
  On pressing buttons related to work modes, it changes all needed variables to switch it.
  Also it controls preventors, locks and manual mode turning on/off.
  Each part of module marked with short description.
  */

  } else if(isSettingsMenuActive==false&&isErrorActive==false&&isSettingsMenuWelcomePageActive==false){

    //Switching work mode to stop. Highest priority.

     if(getButtonState(STOP_BUTTON)==0){
      menuWorkMode = STOPMODE; 
      isLockedForShuttingDown=false;
      isAUTOactive=false;
      turnOnOnlyOnce = true;
    }else {
      if(isLockedForTurningOn==false){

        //Turning on in manual mode.

        if(getButtonState(MANUAL_BUTTON)==0){ 
          menuWorkMode = MANUALMODE;
          isAUTOactive=false;
          if( turnOnOnlyOnce==true){
            lockShuttingDown(5000);
          }
          turnOnOnlyOnce = false;

        //Turning on in auto mode.

         } else if(getButtonState(AUTO_BUTTON)==0){ 
          if(digitalRead(SUPPLYCONTROLPIN) == supplyTurningOn){
            menuWorkMode = AUTOMODE;}
          isAUTOactive=true;
          if( turnOnOnlyOnce==true){
            lockShuttingDown(5000);
          }
          turnOnOnlyOnce = false;
        }
      }
    }

  //turning manual mode off. Almost same priority with stopping.
    if(menuWorkMode==MANUALMODE&&getButtonState(MANUAL_BUTTON)==1){
      menuWorkMode=STOPMODE; 
      isAUTOactive=false;
      isLockedForShuttingDown=false;
      turnOnOnlyOnce = true;
    }
    menuOutput();
  }

//Module that catches errors.
/*How it works:
If the current work mode is manual, or auto with supply control turned on(line 816)
and if turning-off lock is disabled, it comparises actual power circuit parameters with 
critical ones. If every parameter is within limits, nothing done, in other case error 
happens. It sets work mode to stop, lock turning power supply on and outputs error on screen.
*/

     if(menuWorkMode!=STOPMODE){
     if(isLockedForShuttingDown==false){
  if(amperageINTERIOR>=amperageMAX){
    menuWorkMode=STOPMODE; 
      isAUTOactive=false;
      isLockedForShuttingDown=false;
      turnOnOnlyOnce = true;
      lockTurningOn(10000);
      isNeededUpdate=true;
      errorOnlyOutput(err_amperage_high,5000);
  } else if(amperageINTERIOR<=amperageMIN){
    menuWorkMode=STOPMODE; 
      isAUTOactive=false;
      isLockedForShuttingDown=false;
      turnOnOnlyOnce = true;
      lockTurningOn(10000);
      isNeededUpdate=true;
      errorOnlyOutput(err_amperage_low,5000);
  } else if(voltageINTERIOR<=voltageMIN){
    menuWorkMode=STOPMODE; 
      isAUTOactive=false;
      isLockedForShuttingDown=false;
      turnOnOnlyOnce = true;
      lockTurningOn(10000);
      isNeededUpdate=true;
      errorOnlyOutput(err_voltage_low,5000);
  } else if(voltageINTERIOR>=voltageMAX){
    menuWorkMode=STOPMODE; 
      isAUTOactive=false;
      isLockedForShuttingDown=false;
      turnOnOnlyOnce = true;
      lockTurningOn(10000);
      isNeededUpdate=true;
      errorOnlyOutput(err_voltage_high,5000);
      
  }
}}

//This module controls relay. HIGH - apply power in circuit, LOW - block it.

      switch(menuWorkMode){
        case 1:{//STOP
        digitalWrite(RELAYPIN,LOW);
          break;}
        case 2:{//AUTO
          digitalWrite(RELAYPIN,HIGH);
          break;}
        case 3:{//MANUAL
          digitalWrite(RELAYPIN,HIGH);
          break;}
        default:{ //Preventing accidental power supplying
          digitalWrite(RELAYPIN,LOW);
          break;}}

//This module controls entering settings menu.
/*How it works:
In stopped work mode, if user press SET button for 3 seconds, settings menu opens. 
It means, the main menu is blocked, the settings menu welcome page set to active.
Preventor for not skipping next page, and also preventor set for untouch, 1-2 ms 
lags will not cause restarting timer, and user will press button only for 3 secs 
each time. 
If not stoppped, user will be warned and settings menu won't be opened.
If timer not set, it will be set on pressing SET button.

*/

 if(getButtonState(SET_BUTTON)==0){
  if(!isSettingsMenuActive){
      if(isSettingsMenuWelcomePageActive==false){
        if(menuWorkMode==STOPMODE&&isAUTOactive==false){
          isSetUntouched=0;
          if(isSettingsMenuTimerSet){
          if((millis()-settingsMenuTimer)>=3000){
          isSettingsMenuWelcomePageActive = true;
          isNeededUpdate=true;
          istouchedSET_BUTTONPreventor=true;
      }
      }else if(!isSettingsMenuTimerSet){
        isSettingsMenuTimerSet = true;
        settingsMenuTimer = millis();
 }
 }else if(menuWorkMode!=STOPMODE||isAUTOactive==true){
  errorOnlyOutput(err_not_stopped,500);
 }
 }
 }
 if(getButtonState(SET_BUTTON)==1){
        if(isSetUntouched>=10){
          isSettingsMenuTimerSet = false;}
        else {isSetUntouched+=1; }
 }
  }

//The settings welcome page. Ensures user want to enter settings menu, and passes if yes.

  if(isSettingsMenuWelcomePageActive){
    updateIfNeeded();
    lcd.setCursor(0,0);
    lcd.print("OPEN SETTINGS?");
    lcd.setCursor(0,1);
    lcd.print("SET-YES/EXT-NO");

    //Turning off preventor.

    if(getButtonState(SET_BUTTON)==1){
       istouchedSET_BUTTONPreventor=false;
    }

    // Setting all needed variables to open settings menu, if user want to open it. 

    if(getButtonState(SET_BUTTON)==0&& istouchedSET_BUTTONPreventor==false){
          isSettingsMenuWelcomePageActive = false;
          isSettingsMenuTimerSet=false;
          isSettingsMenuActive = true;
          settingmap = 2;
          isNeededUpdate = true;
           istouchedSET_BUTTONPreventor=true;
        }

    //Other preventor turning off. 

    if(getButtonState(EXIT_BUTTON)==1){
    istouchedEXIT_BUTTONPreventor=false;
  }

    //Exiting settings menu welcome page if user wants it. 

    if(getButtonState(EXIT_BUTTON)==0&&istouchedEXIT_BUTTONPreventor==false){
      istouchedEXIT_BUTTONPreventor=true;
      isNeededUpdate = true;
      isSettingsMenuWelcomePageActive = false;
      isSettingsMenuTimerSet=false;
    }
  }
  
  // The settings menu main page. This module is the non-concrete menu opened.
  /*How it works:
  The function used, is closely related with global variables used here, and this module at all.
  Changing page, going to other layers, choosing concrete setting is realised in function, by
  variable settingmap, that changes in function and call it with other aruments here.
  */

  if(!isConcreteSettingChoosed){
    if(isSettingsMenuActive==true){
      updateIfNeeded();
      switch(settingmap){
        case 1:{
          settingsMapFunc("SETTINGS"," ",6,2,999,0,0,0);
          //delay(100);
          break;
        }
        case 2:{
          settingsMapFunc("    Amperage","    Maximum",5,3,0,1,0,0);
          //delay(100);
          break;
        }
        case 3:{
          settingsMapFunc("    Amperage","    Minimum",2,4,1,1,0,0);
          //delay(100);
          break;
        }
        case 4:{
          settingsMapFunc("    Voltage","    Maximum",3,5,2,1,0,0);
          //delay(100);
          break;
        }
        case 5:{
          settingsMapFunc("    Voltage","    Minimum",4,6,3,1,0,0);
          //delay(100);
          break;
        }
        case 6:{
          settingsMapFunc("Set to default"," ",5,2,999,2,0,7);
          //delay(100);
          break;
        }
        case 7:{
          settingsMapFunc("Are you sure?","SET-YES/EXT-NO",7,7,999,3,6,0);
          //delay(100);
          break;
        }
      }
}

  //The concrete setting menu. Just controls, which one and with which arguments will be called.
    
  }else if(isConcreteSettingChoosed){
    updateIfNeeded();
    switch(choosedConcreteSettingID){
      case 0:{
        concreteSettingsMapFuncFloat(0.05,"I max"," A",99.9,0.0,0);
        break;
      }
      case 1:{
        concreteSettingsMapFuncFloat(0.05,"I min"," A",99.9,0.0,5);
        break;
      }
      case 2:{
        concreteSettingsMapFuncInt(1,"U max"," V",259,0,10);
        break;
      }
      case 3:{
        concreteSettingsMapFuncInt(1,"U min"," V",259,0,15);
        break;
      }
      
    }
  }

  //Lock's timers.
  /*How it works:
    Shutting-down lock just disables after time is gone.
    The turning on lock makes same, but also turning menu 
    in other mode if device restarted and this needed.
  */
  if(isLockedForShuttingDown == true){
    if(millis()-shuttingDownLockTimer>=lockTimerShuttingDownINT){
      isLockedForShuttingDown = false;
      isNeededUpdate=true;
    } 
  }
  if(isLockedForTurningOn == true){
    if(millis()-turningOnLockTimer>=lockTimerTurningOnINT){
      isLockedForTurningOn = false;
      isNeededUpdate=true;
      if(neededToTurnOn==true){
        if(afterlockingmode == true){
          menuWorkMode=AUTOMODE;
          neededToTurnOn = false;
        } else if(afterlockingmode == false){
          menuWorkMode=STOPMODE;
          neededToTurnOn = false;
        }
      }
      
    } 
  } 
}


/*
  Some questionable moments:

    1. Why so many global variables, and none local?

    The dynamic memory of Arduino is quickly runs away, causing troubles. And, I noticed that the main reason of it
    is unoptimized usage of local variables, new ones are creating but old ones is not clearing up as soon as needed.
    The most easy way to fix it - using global variables, and rewrite it when needed, even if local one preffered.

    2. Why not using pointers and references?

    Reason is simple - not needed. Clearing up after functions is better, and this almost not damages script.
    Also, I use Atmel Atmega 328pb against original Arduino Uno, and it causes problems. I don't found reason yet.

    3. Why used "Error" against other words, like "Warning"?

    Word "Error" is very negative, and stimulates people more that something like "Warning" or "Trouble". 
    For example, even in compiler the "Warnings" not stop the code against "Errors".

    4. Why default settings are on this level?

    They are temporary, for testing. Also they allowed by PZEM-004t

    5. Where is the scheme of this device? 

    The scheme is not done, because this is only basical side-version script, specified for shield.
    The testing details are:
    PZEM-004t
    Atmel Atmega328pb (Arduino Uno analog)
    D1ROBOT LCD SHIELD 
    Wires
    Relay(SRD-05VDC-SL-C)  --- to pin D2
    Just contact to apply 5V on it, used to control power supply --- to pin D3
    To simulate supply control, used button connected directly between D3 and 5V, with GND through resistor.

    6. How to contact me to ask some questions:

    Phone number: +380 50 851 79 06
    E-mail: poliakov.vladislav.v@gmail.com
    Telegram: https://t.me/ParrKenna | @ParrKenna | Search by phone opened
    Linkedin: https://www.linkedin.com/in/владислав-поляков-691b28298/
    Djinni: https://djinni.co/q/dbfdf0f6b8/
    Viber by phone +380508517906
    

*/