# sdvx_con_AI2

## How to build the wiring in Wokwi
> can be used for demonstartion, not for actually playing
1. First place out all the components, 2 rotary encoders and 7 buttons (4 BT, 2 FX and start as seen in fig #2)
![fig #1](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_1.png)
![fig #2](https://github.com/Pugking4/sdvx_con_AI2/blob/main/sdvx_button_example.jpg)

2. Add a breadboard and connect the ardiuno's 5V and GND to their associated lower rails, then which ever encoder you want to be VOL-L CLK pin to digital pin 0 (assume all pins from here to be digital), DT to pin 7, SW can be left alone, + (power) to the 5V rail on the breadboard and the GND to the GND rail on the breadboard (ignore the wiring on the top of the breadboard)
![fig #3](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_2.png)

3. Repeat for VOL-R encoder with the pin assignments of CLK to 3, DT to 5, + (power) to bb (breadboard) power rail and GND to bb GND rail.
![fig #4](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_3.png)

4. Place all the buttons on the bb in a line and then attach the 2nd ardiuno GND pin to the upper GND bb rail.
![fig #5](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_4.png)

5. Connect the buttons from left to right; top pin 2 always to top bb GND rail and bottom pin 1 to: A1, A3, 0, A5, 3, and 1
![fig #6](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_5.png)

> This code was made for these pin assignments but will error due to the external libraries, please do not use it in Wokwi,
>  credit to [knuckleslee](https://github.com/knuckleslee/RhythmCodes) for the firmware 
```C++
 /*Arduino SDVX Controller Code for Leonardo
 * 2 Encoders + 8 Buttons + 11 HID controlable LED
 * release page
 * http://knuckleslee.blogspot.com/2018/06/RhythmCodes.html
 * 
 * Arduino Joystick Library
 * https://github.com/MHeironimus/ArduinoJoystickLibrary/
 * mon's Arduino-HID-Lighting
 * https://github.com/mon/Arduino-HID-Lighting
 */
#include <Joystick.h>
Joystick_ Joystick(JOYSTICK_DEFAULT_REPORT_ID,JOYSTICK_TYPE_GAMEPAD, 8, 0,
 true, true, false, false, false, false, false, false, false, false, false);

boolean hidMode, state[2]={false}, set[4]={false};
int encL=0, encR=0;
const int PULSE = 15;  //number of pulses per revolution of encoders 
byte EncPins[]    = {0, 7, 3, 5};
// button sorting = {Start,BT-A,-B,-C,-D,FX-L,-R,Extra}
byte SinglePins[] = {4, 6, 12,18,20,22,14,16};
byte ButtonPins[] = {A1, A3, 10, A5, 13, 11, 10, 9};
byte RGBPins[][3] = {{9,10,11},};  //  color sorting = {{Red,Green,Blue},}
char rgbCommon = '+';  //type of your rgb led common pin
unsigned long ReactiveTimeoutMax = 1000;  //number of cycles before HID falls back to reactive

/* pin assignments
 * VOL-L Green to pin 0 and White to pin 1
 * VOL-R Green to pin 2 and White to pin 3
 * current pin layout
 *  SinglePins {4, 6, 12,18,20,22,14,16} = LED 1 to 8
 *    connect pin to resistor and then + termnial of LED
 *    connect ground to - terminal of LED
 *  ButtonPins {5, 7, 13,19,21,23,15,17} = Button input 1 to 8
 *    connect button pin to ground to trigger button press
 *  RGBPins {9,10,11} = PWM RGB light control signal
 *    connect pins to termnial of RGB LEDs
 *    if you using common anode   LED, connect power  to + terminal of LED and set rgbCommon to '+'
 *    if you using common cathode LED, connect ground to - terminal of LED and set rgbCommon to '-'
 *  Light mode detection by read first button while connecting usb 
 *   hold    = false = reactive lighting 
 *   release = true  = HID lighting with reactive fallback
 */

byte ButtonCount = sizeof(ButtonPins) / sizeof(ButtonPins[0]);
byte SingleCount = sizeof(SinglePins) / sizeof(SinglePins[0]);
byte EncPinCount = sizeof(EncPins) / sizeof(EncPins[0]);
unsigned long ReactiveTimeoutCount = ReactiveTimeoutMax;

int ReportDelay = 700;
unsigned long ReportRate ;

void setup() {
  Serial.begin(9600);
  Joystick.begin(false);
  Joystick.setXAxisRange(-PULSE/2, PULSE/2-1);
  Joystick.setYAxisRange(-PULSE/2, PULSE/2-1);
  
  // setup I/O for pins
  for(int i=0;i<ButtonCount;i++) {
    pinMode(ButtonPins[i],INPUT_PULLUP);
  }
  for(int i=0;i<SingleCount;i++) {
    pinMode(SinglePins[i],OUTPUT);
  }
  for(int i=0;i<EncPinCount;i++) {
    pinMode(EncPins[i],INPUT_PULLUP);
  }

  //setup interrupts
  attachInterrupt(digitalPinToInterrupt(EncPins[0]), doEncoder0, CHANGE);
  attachInterrupt(digitalPinToInterrupt(EncPins[2]), doEncoder1, CHANGE);
  
  // light mode detection
  hidMode = digitalRead(ButtonPins[0]);
  while(digitalRead(ButtonPins[0])==LOW) {
    if ( (millis() % 1000) < 500) {
      for(int i=0;i<SingleCount;i++) {
        digitalWrite(SinglePins[i],HIGH);
      }
    }
    else if ( (millis() % 1000) > 500) {
      for(int i=0;i<SingleCount;i++) {
        digitalWrite(SinglePins[i],LOW);
      }
    }
  }
  for(int i=0;i<SingleCount;i++) {
    digitalWrite(SinglePins[i],LOW);
  }

  //boot light
  int startup[] = {0,1,2,3,4,5,6,5,4,3,2,1,0,1,5,2,0,3,6,4,6,3,0,2,5,1};
  for(int i=0;i<(sizeof(startup) / sizeof(startup[0]));i++){
    digitalWrite(SinglePins[startup[i]],HIGH);
    delay(80);
    digitalWrite(SinglePins[startup[i]],LOW);
  }
  for(int i=0;i<SingleCount ;i++){
    digitalWrite(SinglePins[i],HIGH);
  }
    delay(500);
  for(int i=0;i<SingleCount ;i++){
    digitalWrite(SinglePins[i],LOW);
  }
} //end setup

void loop() {
  ReportRate = micros() ;
  
  // read buttons
  for(int i=0;i<ButtonCount;i++) {
    Serial.println(digitalRead(ButtonPins[1]));
    Joystick.setButton (i,!(digitalRead(ButtonPins[i])));
  }

  if(hidMode==false || (hidMode==true && ReactiveTimeoutCount>=ReactiveTimeoutMax)){
    for(int i=0;i<ButtonCount;i++) {
      digitalWrite (SinglePins[i],!(digitalRead(ButtonPins[i])));
    }
  }
  else if(hidMode==true) {
    ReactiveTimeoutCount++;
  }

  //read encoders, detect overflow and rollover
  if(encL < -PULSE/2 || encL > PULSE/2-1)
  encL = constrain (encL*-1, -PULSE/2, PULSE/2-1);
  if(encR < -PULSE/2 || encR > PULSE/2-1)
  encR = constrain (encR*-1, -PULSE/2, PULSE/2-1);
  Joystick.setXAxis(encL);
  Joystick.setYAxis(encR);

  //report
  Joystick.sendState();
  delayMicroseconds(ReportDelay);
  //ReportRate Display
  //Serial.print(micros() - ReportRate) ;
  //Serial.println(" micro sec per loop") ;
}//end loop

//Interrupts
void doEncoder0() {
  if(state[0] == false && digitalRead(EncPins[0]) == LOW) {
    set[0] = digitalRead(EncPins[1]);
    state[0] = true;
  }
  if(state[0] == true && digitalRead(EncPins[0]) == HIGH) {
    set[1] = !digitalRead(EncPins[1]);
    if(set[0] == true && set[1] == true) {
      encL++;
    }
    if(set[0] == false && set[1] == false) {
      encL--;
    }
    state[0] = false;
  }
}

void doEncoder1() {
  if(state[1] == false && digitalRead(EncPins[2]) == LOW) {
    set[2] = digitalRead(EncPins[3]);
    state[1] = true;
  }
  if(state[1] == true && digitalRead(EncPins[2]) == HIGH) {
    set[3] = !digitalRead(EncPins[3]);
    if(set[2] == true && set[3] == true) {
      encR++;
    }
    if(set[2] == false && set[3] == false) {
      encR--;
    }
    state[1] = false;
  }
}
```

## Build controller using 3d printed components
> Can be used to play Unnamed SDVX Clone (USC)
1. Gather all the components which include: 2x rotary encoders, 7x pushbuttons with soldering legs, wires, soldering iron, solder, leonardo ardiuno, 3d printed case and a few nuts which would have been included with the components



## Build custom PCB in KiCad
> Theoretically could be used instead of a leonardo to play USC with the current code
1. First open the KiCad Schematic Editor and add the ATmega32U4-A microcontroller using the add symbol button
<img width="1280" alt="fig #6" src="https://github.com/Pugking4/sdvx_con_AI2/assets/24513580/381e913f-de39-4018-9505-ee0254e582cf">

2. Now add all the components we'll be using which are 7x "Conn_01x02_Socket", 2x "Conn_01x04_Socket", 1x "Conn_01x06_Socket", 2x "R" and 1x "GND". Note the placing is important.
![image](https://github.com/Pugking4/sdvx_con_AI2/assets/24513580/92f539ca-ab18-4a2a-ab17-9179d0197352)

3. Now connect all the 01x02 connectors on pin 2 to the microcontroller GND pin and on pin 1 starting from J1 - j7: 26, 25, 38, 39, 40, 41 and 1, also connect the microcontroller GND pin to the common GNF symbol. Furthermore, connect the J8 01x04 connector from pin 1 - 4: 15, 14, 21 and 18. Also connect the J9 01x04 from 1 - 4 to: 15, 14, 20 and 19. Now connect the J10 from pins 1 - 6 to: 14, 3, 4, 15, to one side of R1 and then from the otherside to 15 and the same for pin 6 but with R2.
![image](https://github.com/Pugking4/sdvx_con_AI2/assets/24513580/30bac342-1a78-4ee6-a09f-230ded8b55d7)




