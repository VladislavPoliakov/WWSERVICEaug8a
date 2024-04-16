/*
Made by V.V.Poliakov (ParrKenna) for WWSERVICE.
Comments are for following code, that goes after it:
//COMMENT
    |
    V
code();
Some questionable moments and my contacts are listed and explained in the end of this code/document.
*/
// Libraries.
#include <Arduino.h>
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
// Buttons on the shield.
#define BUTTON_RIGHT 0
#define BUTTON_UP 1
#define BUTTON_DOWN 2
#define BUTTON_LEFT 3
#define BUTTON_SELECT 4
#define ANY_BUTTON  5
// Pins on shield.
#define RELAY_PIN 2
#define SUPPLY_CONTROL_PIN 3
#define KEYBOARD_PIN A0
// Work modes.
#define STOP_MODE 1
#define AUTO_MODE 2
#define MANUAL_MODE 3
/*
 DEFAULT values of settings.
 Used ONLY for debug,
 setting to default and
 setting up new device.
*/
#define AMPERAGE_MAXIMUM_DEFAULT 7.95
#define AMPERAGE_MINIMUM_DEFAULT 3.95
#define VOLTAGE_MAXIMUM_DEFAULT 245
#define VOLTAGE_MINIMUM_DEFAULT 195
#define WATTAGE_MULTIPLIER_DEFAULT 0.9
// Error codes.
#define ERROR_NOT_STOPPED 1
#define ERROR_AMPERAGE_TOO_HIGH 2
#define ERROR_AMPERAGE_TOO_LOW 3
#define ERROR_VOLTAGE_TOO_LOW 4
#define ERROR_VOLTAGE_TOO_HIGH 5
// Custom symbols for the LCD
byte g_locker_symbol[8]{
    0b00100,
    0b01010,
    0b01010,
    0b01010,
    0b11111,
    0b11011,
    0b11011,
    0b01110,
};
byte g_turning_on_mode_symbol[8]{
    0b00000,
    0b00000,
    0b00100,
    0b10101,
    0b10101,
    0b10001,
    0b01110,
    0b00000,
};
// Inititalizing of LCD and PZEM-004t
LiquidCrystal lcd(8, 9, 4, 5, 6, 7);
PZEM004Tv30 pzem(Serial);
// Comparizing with current signal of Supply control pin.
short int supply_turning_on = 1;
// Locks&clocks for starting and stopping
bool g_is_locked_for_turning_on;
bool g_is_locked_for_shutting_down;
unsigned int g_lock_turning_on_delay;
unsigned int g_lock_shutting_down_delay;
uint32_t g_turning_on_lock_timer;
uint32_t g_shutting_down_lock_timer;
/*
 Function used when error occured.
 Locking turning on, for the milliseconds that is defined by fv_set_timer variable.
*/
void f_lock_turning_on(int fv_set_timer)
{
  // Lock.
  g_is_locked_for_turning_on = true;
  // Count current time.
  g_turning_on_lock_timer = millis();
  // Setting or updating current delay.
  g_lock_turning_on_delay = fv_set_timer;
}
/*
 Function used when the AUTO or MANUAL mode is only started.
 Locking turning on, for the milliseconds that is defined by fv_set_timer variable.
 Locking shutting down by the error catcher.
 User can shut down device manually, it can't be locked.
*/
void f_lock_shutting_down(int fv_set_timer)
{
  // Locking.
  g_is_locked_for_shutting_down = true;
  // Count current time.
  g_shutting_down_lock_timer = millis();
  // Setting or updating current delay.
  g_lock_shutting_down_delay = fv_set_timer;
}
/*Function that is used to get the current state of concrete or any button.
Transforms analog signal of keyboard to digital state (0 or 1) of buttons.
1 is not touched.
0 is touched.
 */
bool f_get_button_state(int fv_button)
{
  // Check for button ID.
  if (fv_button == ANY_BUTTON )
  {
    /*
     If button ID is 5(Any button is needed to be touched),
     check the current keyboard signal. If ANY button touched,
     it could be less than 1000.*/
    if (analogRead(KEYBOARD_PIN) > 1000)
    {
      return 1;
    }
    else
    {
      return 0;
    }
  }
  else
  {
    // If button id is concrete, creating variable that will save ID of tuched button.
    int l_real_pressed_button;
    // Getting signal of keyboard.
    int l_keyboard_signal = analogRead(KEYBOARD_PIN);
    // Comparising the signal with buttons signal area map. The current is for D1ROBOT
    if (l_keyboard_signal < 50)
    {
      l_real_pressed_button = BUTTON_RIGHT;
    }
    else if (l_keyboard_signal < 250)
    {
      l_real_pressed_button = BUTTON_UP;
    }
    else if (l_keyboard_signal < 450)
    {
      l_real_pressed_button = BUTTON_DOWN;
    }
    else if (l_keyboard_signal < 650)
    {
      l_real_pressed_button = BUTTON_LEFT;
    }
    else if (l_keyboard_signal < 850)
    {
      l_real_pressed_button = BUTTON_SELECT;
    }
    else
    {
      return 1;
    }
    // Check for the according currently pressed button and button, that is checking for be touched.
    if (fv_button == l_real_pressed_button)
    {
      return 0;
    }
    else
    {
      return 1;
    }
  }
}
// Variables for saving the current and previous(postfix "prev") values got from PZEM-004t
int g_voltage_current_value = 000;
float g_amperage_current_value = 0.00;
float g_wattage_current_value = 0.00;
int g_voltage_previous_value = 000;
float g_amperage_previous_value = 0.00;
float g_wattage_previous_value = 0.00;
// Variable to save the currently active work mode. Work modes IDs defined at line 54.
short int g_menu_and_work_mode= STOP_MODE;
// Function that saving current mode to EEPROM. Saves only STOP or AUTO modes.
void f_write_work_mode_to_EEPROM(bool fv_mode_to_write)
{
  EEPROM.put(50, fv_mode_to_write);
}
// Getting mode to start from EEPROM.
bool f_check_current_mode_for_setup()
{
  bool l_what_to_return;
  EEPROM.get(50, l_what_to_return);
  return l_what_to_return;
}
// Variables to control the settings menu. 0 NO 1 YES.
//Settings menu status.
bool g_is_settings_menu_active = false;
// Variable used to save current settings menu page opened.
int g_setting_map_current_position;
// Variable used to keep opened or closed concrete setting page.
bool g_is_concrete_setting_chosen = false;
/* Variable used analogically to g_setting_map_current_position,
but only for settings that have concrete page.*/
unsigned int g_chosen_concrete_setting_ID;
// Current settings. Used to catch errors.
float g_amperage_maximum;
float g_amperage_minimum;
int g_voltage_maximum;
int g_voltage_minimum;
float g_wattage_multiplier;
// Temporary values for settings, to change them.
float g_temporary_setting_float;
int g_temporary_setting_int;
// Variables used in error catch system.
uint32_t g_error_output_disable_timer;
bool g_is_error_active = false;
unsigned int g_saved_error_code;
unsigned int g_saved_disable_delay;
// Variable for screen updater. Used to check if update screen is needed.
bool is_update_needed = 1;
// Screen updater. Checks if needed to clear it, and does it if yes.
void f_update_if_needed()
{
  if (is_update_needed)
  {
    lcd.clear();
    is_update_needed = false;
  }
}
// Exit from setting menu and clearing after.
bool g_is_auto_mode_active = false;
void f_exit_from_settings_menu()
{
  is_update_needed = true;
  g_is_settings_menu_active = 0;
  g_menu_and_work_mode= STOP_MODE;
  g_is_auto_mode_active = false;
}
// Getting settings from EEPROM to RAM.
void f_get_settings_from_EEPROM()
{
  EEPROM.get(0, g_amperage_maximum);
  EEPROM.get(5, g_amperage_minimum);
  EEPROM.get(10, g_voltage_maximum);
  EEPROM.get(15, g_voltage_minimum);
  EEPROM.get(20, g_wattage_multiplier);
}
// Setting to default. Default setting on line 60. Accompanied with visual effects.
void f_set_to_default_settings_to_EEPROM()
{
  is_update_needed = true;
  f_update_if_needed();
  lcd.setCursor(0, 0);
  lcd.print("SETTING TO");
  lcd.setCursor(0, 1);
  lcd.print("DEFAULT");
  EEPROM.put(0, AMPERAGE_MAXIMUM_DEFAULT);
  EEPROM.put(5, AMPERAGE_MINIMUM_DEFAULT);
  EEPROM.put(10, VOLTAGE_MAXIMUM_DEFAULT);
  EEPROM.put(15, VOLTAGE_MINIMUM_DEFAULT);
  EEPROM.put(20, WATTAGE_MULTIPLIER_DEFAULT);
  delay(100);
  lcd.print(" .");
  delay(250);
  lcd.print(" .");
  delay(250);
  lcd.print(" .");
  delay(500);
  is_update_needed = true;
  f_update_if_needed();
  lcd.setCursor(0, 0);
  lcd.print("DONE");
  delay(3000);
}
// Visual part and activity setter of Error catching module. Compares Error codes and updates screen.
void f_error_only_output(unsigned int fv_error_code, int fv_disable_delay)
{
  g_saved_error_code = fv_error_code;
  g_saved_disable_delay = fv_disable_delay;
  // Activity setter.
  if (g_is_error_active == false)
  {
    g_error_output_disable_timer = millis();
    g_is_error_active = true;
  }
  f_update_if_needed();
  lcd.setCursor(0, 0);
  // Following part is used in every error except "not stopped", on top line of LCD.
  if (fv_error_code!= 1)
  {
    lcd.print("ERROR");
  }
  // Prints current error description on bottom line of LCD, except first.
  lcd.setCursor(0, 1);
  switch (fv_error_code)
  {
  case ERROR_NOT_STOPPED:
  {
    // Prints "NOT STOPPED" on top line of LCD.
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("NOT STOPPED");
    break;
  }
    // Any other prints "ERROR" in top line and current error on bottom line.
  case ERROR_AMPERAGE_TOO_HIGH:
  {
    lcd.print("AMPERAGE HIGH");
    break;
  }
  case ERROR_AMPERAGE_TOO_LOW:
  {
    lcd.print("AMPERAGE LOW");
    break;
  }
  case ERROR_VOLTAGE_TOO_LOW:
  {
    lcd.print("VOLTAGE LOW");
    break;
  }
  case ERROR_VOLTAGE_TOO_HIGH:
  {
    lcd.print("VOLTAGE HIGH");
    break;
  }
  }
}
// The home page visual
void f_menu_output()
{
  // ONLY FOR TEST
  // The test used for check keyboard.
  /* if(f_get_button_state(STOP_BUTTON)){
     lcd.setCursor(0,0);
     lcd.print("stop");
   }else if(f_get_button_state(PLUS_BUTTON)){
     lcd.setCursor(0,0);
     lcd.print("+/auto");
   } else if(f_get_button_state(MINUS_BUTTON)){
     lcd.setCursor(0,0);
     lcd.print("-/man");
   }else if(f_get_button_state(EXIT_BUTTON)){
     lcd.setCursor(0,0);
     lcd.print("EXT");
   }else if(f_get_button_state(SET_BUTTON)){
     lcd.setCursor(0,0);
     lcd.print("set");
   }*/
  // ONLY FOR TEST
  // Checking current parameters, comparizing with previous. If found diffent, clears screen.
  if (g_voltage_current_value != g_voltage_previous_value)
  {
    is_update_needed = true;
  }
  else if (g_amperage_current_value != g_amperage_previous_value)
  {
    is_update_needed = true;
  }
  else if (g_wattage_current_value != g_wattage_previous_value)
  {
    is_update_needed = true;
  }
  f_update_if_needed();
  // Printing information to screen. It is a home page of device, used most time.
  lcd.setCursor(0, 0);
  lcd.print("U:");
  lcd.print(g_voltage_current_value);
  lcd.print("V");
  lcd.setCursor(8, 0);
  lcd.print("I:");
  lcd.print(g_amperage_current_value, 2);
  lcd.print("A");
  lcd.setCursor(0, 1);
  lcd.print("P:");
  lcd.print(g_wattage_current_value, 2);
  lcd.print("kW");
  lcd.setCursor(10, 1);
  // Printing custom symbols to show the turning on and shutting down locks.
  if (g_is_locked_for_turning_on == true)
  {
    lcd.print(char(1));
  }
  else if (g_is_locked_for_shutting_down == true)
  {
    lcd.print(char(2));
  }
  else
  {
    lcd.print(" ");
  }
  // Printing current work mode on screen.
  lcd.setCursor(12, 1);
  switch (g_menu_and_work_mode)
  {
  case STOP_MODE:
  {
    if (g_is_auto_mode_active == false)
    {
      lcd.print("STOP");
    }
    else
    {
      lcd.print("AUTO");
    }
    break;
  }
  case AUTO_MODE:
  {
    lcd.print("AUTO");
    break;
  }
  case MANUAL_MODE:
  {
    lcd.print("MAN.");
    break;
  }
  }
  // Saving current parameters for next comparison.
  g_voltage_previous_value = g_voltage_current_value;
  g_amperage_previous_value = g_amperage_current_value;
  g_wattage_previous_value = g_wattage_current_value;
}
// Variable to save mode which will be saved to and loaded from EEPROM, and preventors for correct buttons work.
bool g_mode_after_locking;
bool g_is_touched_set_button_preventor = false;
bool g_is_touched_exit_button_preventor = false;
// Function for settings menu, first level - user can choose which setting will be changed.
void f_settings_map_function(const String &fv_print_1, const String &fv_print_2, unsigned short int fv_up, unsigned short int fv_down, unsigned short int fv_chosen_concrete_setting_ID, unsigned short int fv_set_choise, unsigned short int fv_exit_destination_from_layer_2, unsigned short int fv_set_layer_2)
{
  // Printing current setting in list
  f_update_if_needed();
  lcd.setCursor(0, 0);
  lcd.print(fv_print_1);
  lcd.setCursor(0, 1);
  lcd.print(fv_print_2);
  // Ensures that user unpressed buttons before next moves.
  if (f_get_button_state(SET_BUTTON) == 1)
  {
    g_is_touched_set_button_preventor = false;
  }
  if (f_get_button_state(EXIT_BUTTON) == 1)
  {
    g_is_touched_exit_button_preventor = false;
  }
  // delay(250);
  // Menu navigation, to next and previous setting to choose.
  if (f_get_button_state(PLUS_BUTTON) == 0)
  {
    is_update_needed = true;
    g_setting_map_current_position = fv_up;
    // delay(250);
  }
  else if (f_get_button_state(MINUS_BUTTON) == 0)
  {
    is_update_needed = true;
    g_setting_map_current_position = fv_down;
    // delay(250);
    // Menu navigation to exit from setting to main screen.
  }
  else if (f_get_button_state(EXIT_BUTTON) == 0 && g_is_touched_exit_button_preventor == false)
  {
    g_is_touched_exit_button_preventor = true;
    if (fv_exit_destination_from_layer_2 == 0)
    {
      is_update_needed = true;
      f_exit_from_settings_menu();
      // delay(250);
      // Also, for simulation of second layer.
    }
    else if (fv_exit_destination_from_layer_2 != 0)
    {
      is_update_needed = true;
      g_setting_map_current_position = fv_exit_destination_from_layer_2;
      // delay(250);
    }
    // delay(250);
    // Menu navigation to choose concrete setting to change.
  }
  else if (f_get_button_state(SET_BUTTON) == 0 && g_is_touched_set_button_preventor == false)
  {
    g_is_touched_set_button_preventor = true;
    g_is_touched_exit_button_preventor = false;
    // If 0, do nothing. Used in case of "SETTINGS" welcome page.
    if (fv_set_choise == 0)
    {
      is_update_needed = true;
      // If 1, user choised concrete setting.
    }
    else if (fv_set_choise == 1)
    {
      g_is_touched_set_button_preventor = true;
      g_chosen_concrete_setting_ID = fv_chosen_concrete_setting_ID;
      is_update_needed = true;
      g_is_concrete_setting_chosen = true;
      // In this case, used uneffective method, but more safe than using unspecified pointers.
      // Saving user choice.
      switch (fv_chosen_concrete_setting_ID)
      {
      case 0:
      {
        g_temporary_setting_float = g_amperage_maximum;
        break;
      }
      case 1:
      {
        g_temporary_setting_float = g_amperage_minimum;
        break;
      }
      case 2:
      {
        g_temporary_setting_int = g_voltage_maximum;
        break;
      }
      case 3:
      {
        g_temporary_setting_int = g_voltage_minimum;
        break;
      }
      case 4:
      {
        g_temporary_setting_float = g_wattage_multiplier;
        break;
      }
      }
      is_update_needed = true;
      // If 2, used simulation of second layer.
    }
    else if (fv_set_choise == 2)
    {
      is_update_needed = true;
      g_setting_map_current_position = fv_set_layer_2;
      // delay(500);
      // Case 3 used only for setting device to default, and first usage.
    }
    else if (fv_set_choise == 3)
    {
      f_set_to_default_settings_to_EEPROM();
      g_setting_map_current_position = 1;
    }
    // delay(500);
  }
}
// Function for checking if type is integer or not.
// In any case it returns false.
template <typename T>
bool f_is_integer(T)
{
  return false;
}
// But by respecifying the template, it returns true when type is integer.
template <>
bool f_is_integer(int)
{
  return true;
}
// Function for concrete setting change menu, of parameter that is saved as integer.
// using template to create ablitity to work with both int and float types in same function.
template <typename T>
void f_concrete_settings_map_function(T fv_change_scale, const String &fv_print_upper_row, const String &fv_measurement_units, T fv_upper_bound, T fv_lower_bound, int fv_EEPROM_adress)
{
  T l_temporary_setting;
  f_update_if_needed();
  lcd.setCursor(0, 0);
  lcd.print(fv_print_upper_row);
  lcd.setCursor(0, 1);
  if (f_is_integer(l_temporary_setting))
  {
    l_temporary_setting = g_temporary_setting_int;
  }
  else
    l_temporary_setting = g_temporary_setting_float;
  lcd.print(l_temporary_setting);
  lcd.print(fv_measurement_units);
  // delay(1000);
  if (f_get_button_state(SET_BUTTON) == 1)
  {
    g_is_touched_set_button_preventor = false;
  }
  if (f_get_button_state(EXIT_BUTTON) == 1)
  {
    g_is_touched_exit_button_preventor = false;
  }
  if (f_get_button_state(PLUS_BUTTON) == 0)
  {
    if (l_temporary_setting < fv_upper_bound)
    {
      l_temporary_setting += fv_change_scale;
    }
    else
    {
      l_temporary_setting = fv_upper_bound;
      lcd.setCursor(15, 1);
      lcd.print("!");
    }
    is_update_needed = true;
    // delay(250);
  }
  else if (f_get_button_state(MINUS_BUTTON) == 0)
  {
    if (l_temporary_setting > fv_lower_bound)
    {
      l_temporary_setting -= fv_change_scale;
    }
    else
    {
      l_temporary_setting = fv_lower_bound;
      lcd.setCursor(15, 1);
      lcd.print("!");
    }
    is_update_needed = true;
    // delay(250);
  }
  else if (f_get_button_state(EXIT_BUTTON) == 0 && g_is_touched_exit_button_preventor == false)
  {
    g_is_touched_exit_button_preventor = true;
    g_is_concrete_setting_chosen = false;
    is_update_needed = true;
    // delay(250);
  }
  else if (f_get_button_state(SET_BUTTON) == 0 && g_is_touched_set_button_preventor == false)
  {
    // delay(4000);
    g_is_touched_set_button_preventor = true;
    g_is_touched_exit_button_preventor = false;
    is_update_needed = true;
    f_update_if_needed();
    lcd.setCursor(0, 0);
    lcd.print("SAVING");
    EEPROM.put(fv_EEPROM_adress, l_temporary_setting);
    delay(250);
    lcd.print(" .");
    delay(250);
    lcd.print(" .");
    delay(250);
    lcd.print(" .");
    delay(250);
    is_update_needed = true;
    f_update_if_needed();
    lcd.setCursor(0, 0);
    lcd.print("DONE");
    delay(3000);
    is_update_needed = true;
  }
}
// Preventor that will ensure the user seen the error and it's reason.
bool g_is_button_touched_after_error_disabling = false;
void f_error_stop_catcher()
{
  // The preventor will cannot be disabling during little time.
  if (millis() - g_error_output_disable_timer >= g_saved_disable_delay)
  {
    if (g_is_button_touched_after_error_disabling == false)
    {
      if (f_get_button_state(ANY_BUTTON ) == 0)
      {
        g_is_button_touched_after_error_disabling = true;
      }
      // Also, ensuring preventor will be closed after.
    }
    else
    {
      g_is_error_active = false;
      is_update_needed = true;
    }
  }
}
// Timer, and preventors. Needed to manage settings menu.
uint32_t g_settings_menu_opening_timer;
bool g_is_settings_menu_opening_timer_set = false;
bool g_is_settings_menu_welcome_page_active = false;
unsigned short int g_is_set_button_unpressed = 0;
// Variable to turn on device after starting up, if needed.
bool g_is_needed_to_turn_on;
// Setup is called once at start of devices work.
void setup()
{
  // Starting up the LCD, and UART for PZEM-004t
  lcd.begin(16, 2);
  Serial.begin(9600);
  // Turning on device if needed
  if (f_check_current_mode_for_setup() == true)
  {
    g_mode_after_locking = true;
    g_is_needed_to_turn_on = true;
    f_lock_shutting_down(5000);
    g_is_auto_mode_active = true;
  }
  else
  {
    g_mode_after_locking = false;
    g_is_needed_to_turn_on = true;
    g_is_auto_mode_active = false;
  }
  // Starting up pins, LCD's cursor, settings of device.
  pinMode(RELAY_PIN, OUTPUT);         // RELAY
  pinMode(KEYBOARD_PIN, INPUT);        // KEYBOARD
  pinMode(SUPPLY_CONTROL_PIN, INPUT); // SUPPLY CONTROL
  lcd.setCursor(0, 0);
  f_get_settings_from_EEPROM();
  // Loading custom symbols to LCD, end of starting up.
  lcd.createChar(1, g_locker_symbol);
  lcd.createChar(2, g_turning_on_mode_symbol);
  f_lock_turning_on(5000);
  g_is_settings_menu_opening_timer_set = false;
}
// Controller to lock shutting down for start of work.
bool g_turning_on_only_once_preventor = true;
// This function is repeating during all devices work time.
void loop()
{
  // Read and check info from PZEM-004t
  g_voltage_current_value = pzem.voltage();
  g_amperage_current_value = pzem.current();
  g_wattage_current_value = pzem.power() / 1000;
  if (isnan(g_voltage_current_value))
  {
    g_voltage_current_value = 0;
  }
  if (isnan(g_amperage_current_value))
  {
    g_amperage_current_value = 0;
  }
  if (isnan(g_wattage_current_value))
  {
    g_wattage_current_value = 0;
  }
  // Main menu. If stopped or auto turned on, writes it to EEPROM.
  if (g_menu_and_work_mode== STOP_MODE)
  {
    if (f_check_current_mode_for_setup() == true && g_is_auto_mode_active == false)
    {
      f_write_work_mode_to_EEPROM(false);
    }
  }
  else if (g_menu_and_work_mode== AUTO_MODE || g_is_auto_mode_active == true)
  {
    if (f_check_current_mode_for_setup() == false)
    {
      f_write_work_mode_to_EEPROM(true);
    }
  }
  // Specifical requirement from customer:
  /* There must be supply control pin
  When the auto mode is turned on, it must control the power supply.
  When applying power to pin, it must apply power throw relay,
  if no - relay must block power.*/
  /*How it works: almost all logic is checking one of two variables to check activity of auto mode:
  The g_menu_and_work_modeand g_is_auto_mode_active. First variable is control of power supply, monitoring and else
  general functions, second - specifical functions like saving workmode, visual, etc.
  This module controlling the g_menu_and_work_mode, but saving AUTO as active.*/
  f_update_if_needed();
  if (g_is_auto_mode_active == true)
  {
    if (digitalRead(SUPPLY_CONTROL_PIN) == supply_turning_on)
    {
      g_menu_and_work_mode= AUTO_MODE;
      g_is_locked_for_turning_on = false;
      if (g_turning_on_only_once_preventor == true)
      {
        f_lock_shutting_down(5000);
      }
      g_turning_on_only_once_preventor = false;
    }
    else if (digitalRead(SUPPLY_CONTROL_PIN) != supply_turning_on)
    {
      g_menu_and_work_mode= STOP_MODE;
      g_is_locked_for_shutting_down = false;
      g_is_locked_for_turning_on = false;
      g_turning_on_only_once_preventor = true;
    }
  }
  // This prevents next code during errors.
  if (g_is_error_active == true)
  {
    f_error_stop_catcher();
    // If no errors are active at the moment, the next code will control work modes.
    /*How it works:
    On pressing buttons related to work modes, it changes all needed variables to switch it.
    Also it controls preventors, locks and manual mode turning on/off.
    Each part of module marked with short description.
    */
  }
  else if (g_is_settings_menu_active == false && g_is_error_active == false && g_is_settings_menu_welcome_page_active == false)
  {
    // Switching work mode to stop. Highest priority.
    if (f_get_button_state(STOP_BUTTON) == 0)
    {
      g_menu_and_work_mode= STOP_MODE;
      g_is_locked_for_shutting_down = false;
      g_is_auto_mode_active = false;
      g_turning_on_only_once_preventor = true;
    }
    else
    {
      if (g_is_locked_for_turning_on == false)
      {
        // Turning on in manual mode.
        if (f_get_button_state(MANUAL_BUTTON) == 0)
        {
          g_menu_and_work_mode= MANUAL_MODE;
          g_is_auto_mode_active = false;
          if (g_turning_on_only_once_preventor == true)
          {
            f_lock_shutting_down(5000);
          }
          g_turning_on_only_once_preventor = false;
          // Turning on in auto mode.
        }
        else if (f_get_button_state(AUTO_BUTTON) == 0)
        {
          if (digitalRead(SUPPLY_CONTROL_PIN) == supply_turning_on)
          {
            g_menu_and_work_mode= AUTO_MODE;
          }
          g_is_auto_mode_active = true;
          if (g_turning_on_only_once_preventor == true)
          {
            f_lock_shutting_down(5000);
          }
          g_turning_on_only_once_preventor = false;
        }
      }
    }
    // turning manual mode off. Almost same priority with stopping.
    if (g_menu_and_work_mode== MANUAL_MODE && f_get_button_state(MANUAL_BUTTON) == 1)
    {
      g_menu_and_work_mode= STOP_MODE;
      g_is_auto_mode_active = false;
      g_is_locked_for_shutting_down = false;
      g_turning_on_only_once_preventor = true;
    }
    f_menu_output();
  }
  // Module that catches errors.
  /*How it works:
  If the current work mode is manual, or auto with supply control turned on(line 816)
  and if turning-off lock is disabled, it comparises actual power circuit parameters with
  critical ones. If every parameter is within limits, nothing done, in other case error
  happens. It sets work mode to stop, lock turning power supply on and outputs error on screen.
  */
  if (g_menu_and_work_mode!= STOP_MODE)
  {
    if (g_is_locked_for_shutting_down == false)
    {
      if (g_amperage_current_value >= g_amperage_maximum)
      {
        g_menu_and_work_mode= STOP_MODE;
        g_is_auto_mode_active = false;
        g_is_locked_for_shutting_down = false;
        g_turning_on_only_once_preventor = true;
        f_lock_turning_on(10000);
        is_update_needed = true;
        f_error_only_output(ERROR_AMPERAGE_TOO_HIGH, 5000);
      }
      else if (g_amperage_current_value <= g_amperage_minimum)
      {
        g_menu_and_work_mode= STOP_MODE;
        g_is_auto_mode_active = false;
        g_is_locked_for_shutting_down = false;
        g_turning_on_only_once_preventor = true;
        f_lock_turning_on(10000);
        is_update_needed = true;
        f_error_only_output(ERROR_AMPERAGE_TOO_LOW, 5000);
      }
      else if (g_voltage_current_value <= g_voltage_minimum)
      {
        g_menu_and_work_mode= STOP_MODE;
        g_is_auto_mode_active = false;
        g_is_locked_for_shutting_down = false;
        g_turning_on_only_once_preventor = true;
        f_lock_turning_on(10000);
        is_update_needed = true;
        f_error_only_output(ERROR_VOLTAGE_TOO_LOW, 5000);
      }
      else if (g_voltage_current_value >= g_voltage_maximum)
      {
        g_menu_and_work_mode= STOP_MODE;
        g_is_auto_mode_active = false;
        g_is_locked_for_shutting_down = false;
        g_turning_on_only_once_preventor = true;
        f_lock_turning_on(10000);
        is_update_needed = true;
        f_error_only_output(ERROR_VOLTAGE_TOO_HIGH, 5000);
      }
    }
  }
  // This module controls relay. HIGH - apply power in circuit, LOW - block it.
  switch (g_menu_and_work_mode)
  {
  case 1:
  { // STOP
    digitalWrite(RELAY_PIN, LOW);
    break;
  }
  case 2:
  { // AUTO
    digitalWrite(RELAY_PIN, HIGH);
    break;
  }
  case 3:
  { // MANUAL
    digitalWrite(RELAY_PIN, HIGH);
    break;
  }
  default:
  { // Preventing accidental power supplying
    digitalWrite(RELAY_PIN, LOW);
    break;
  }
  }
  // This module controls entering settings menu.
  /*How it works:
  In stopped work mode, if user press SET button for 3 seconds, settings menu opens.
  It means, the main menu is blocked, the settings menu welcome page set to active.
  Preventor for not skipping next page, and also preventor set for untouch, 1-2 ms
  lags will not cause restarting timer, and user will press button only for 3 secs
  each time.
  If not stoppped, user will be warned and settings menu won't be opened.
  If timer not set, it will be set on pressing SET button.
  */
  if (f_get_button_state(SET_BUTTON) == 0)
  {
    if (!g_is_settings_menu_active)
    {
      if (g_is_settings_menu_welcome_page_active == false)
      {
        if (g_menu_and_work_mode== STOP_MODE && g_is_auto_mode_active == false)
        {
          g_is_set_button_unpressed = 0;
          if (g_is_settings_menu_opening_timer_set)
          {
            if ((millis() - g_settings_menu_opening_timer) >= 3000)
            {
              g_is_settings_menu_welcome_page_active = true;
              is_update_needed = true;
              g_is_touched_set_button_preventor = true;
            }
          }
          else if (!g_is_settings_menu_opening_timer_set)
          {
            g_is_settings_menu_opening_timer_set = true;
            g_settings_menu_opening_timer = millis();
          }
        }
        else if (g_menu_and_work_mode!= STOP_MODE || g_is_auto_mode_active == true)
        {
          f_error_only_output(ERROR_NOT_STOPPED, 500);
        }
      }
    }
    if (f_get_button_state(SET_BUTTON) == 1)
    {
      if (g_is_set_button_unpressed >= 10)
      {
        g_is_settings_menu_opening_timer_set = false;
      }
      else
      {
        g_is_set_button_unpressed += 1;
      }
    }
  }
  // The settings welcome page. Ensures user want to enter settings menu, and passes if yes.
  if (g_is_settings_menu_welcome_page_active)
  {
    f_update_if_needed();
    lcd.setCursor(0, 0);
    lcd.print("OPEN SETTINGS?");
    lcd.setCursor(0, 1);
    lcd.print("SET-YES/EXT-NO");
    // Turning off preventor.
    if (f_get_button_state(SET_BUTTON) == 1)
    {
      g_is_touched_set_button_preventor = false;
    }
    // Setting all needed variables to open settings menu, if user want to open it.
    if (f_get_button_state(SET_BUTTON) == 0 && g_is_touched_set_button_preventor == false)
    {
      g_is_settings_menu_welcome_page_active = false;
      g_is_settings_menu_opening_timer_set = false;
      g_is_settings_menu_active = true;
      g_setting_map_current_position = 2;
      is_update_needed = true;
      g_is_touched_set_button_preventor = true;
    }
    // Other preventor turning off.
    if (f_get_button_state(EXIT_BUTTON) == 1)
    {
      g_is_touched_exit_button_preventor = false;
    }
    // Exiting settings menu welcome page if user wants it.
    if (f_get_button_state(EXIT_BUTTON) == 0 && g_is_touched_exit_button_preventor == false)
    {
      g_is_touched_exit_button_preventor = true;
      is_update_needed = true;
      g_is_settings_menu_welcome_page_active = false;
      g_is_settings_menu_opening_timer_set = false;
    }
  }
  // The settings menu main page. This module is the non-concrete menu opened.
  /*How it works:
  The function used, is closely related with global variables used here, and this module at all.
  Changing page, going to other layers, choosing concrete setting is realised in function, by
  variable g_setting_map_current_position, that changes in function and call it with other aruments here.
  */
  if (!g_is_concrete_setting_chosen)
  {
    if (g_is_settings_menu_active == true)
    {
      f_update_if_needed();
      switch (g_setting_map_current_position)
      {
      case 1:
      {
        f_settings_map_function("SETTINGS", " ", 6, 2, 999, 0, 0, 0);
        // delay(100);
        break;
      }
      case 2:
      {
        f_settings_map_function("    Amperage", "    Maximum", 5, 3, 0, 1, 0, 0);
        // delay(100);
        break;
      }
      case 3:
      {
        f_settings_map_function("    Amperage", "    Minimum", 2, 4, 1, 1, 0, 0);
        // delay(100);
        break;
      }
      case 4:
      {
        f_settings_map_function("    Voltage", "    Maximum", 3, 5, 2, 1, 0, 0);
        // delay(100);
        break;
      }
      case 5:
      {
        f_settings_map_function("    Voltage", "    Minimum", 4, 6, 3, 1, 0, 0);
        // delay(100);
        break;
      }
      case 6:
      {
        f_settings_map_function("Set to default", " ", 5, 2, 999, 2, 0, 7);
        // delay(100);
        break;
      }
      case 7:
      {
        f_settings_map_function("Are you sure?", "SET-YES/EXT-NO", 7, 7, 999, 3, 6, 0);
        // delay(100);
        break;
      }
      }
    }
    // The concrete setting menu. Just controls, which one and with which arguments will be called.
  }
  else if (g_is_concrete_setting_chosen)
  {
    f_update_if_needed();
    switch (g_chosen_concrete_setting_ID)
    {
    case 0:
    {
      f_concrete_settings_map_function(0.05, "I max", " A", 99.9, 0.0, 0);
      break;
    }
    case 1:
    {
      f_concrete_settings_map_function(0.05, "I min", " A", 99.9, 0.0, 5);
      break;
    }
    case 2:
    {
      f_concrete_settings_map_function(1, "U max", " V", 259, 0, 10);
      break;
    }
    case 3:
    {
      f_concrete_settings_map_function(1, "U min", " V", 259, 0, 15);
      break;
    }
    }
  }
  // Lock's timers.
  /*How it works:
    Shutting-down lock just disables after time is gone.
    The turning on lock makes same, but also turning menu
    in other mode if device restarted and this needed.
  */
  if (g_is_locked_for_shutting_down == true)
  {
    if (millis() - g_shutting_down_lock_timer >= g_lock_shutting_down_delay)
    {
      g_is_locked_for_shutting_down = false;
      is_update_needed = true;
    }
  }
  if (g_is_locked_for_turning_on == true)
  {
    if (millis() - g_turning_on_lock_timer >= g_lock_turning_on_delay)
    {
      g_is_locked_for_turning_on = false;
      is_update_needed = true;
      if (g_is_needed_to_turn_on == true)
      {
        if (g_mode_after_locking == true)
        {
          g_menu_and_work_mode= AUTO_MODE;
          g_is_needed_to_turn_on = false;
        }
        else if (g_mode_after_locking == false)
        {
          g_menu_and_work_mode= STOP_MODE;
          g_is_needed_to_turn_on = false;
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
    Github: https://github.com/VladislavPoliakov/WWSERVICEaug8a
*/

