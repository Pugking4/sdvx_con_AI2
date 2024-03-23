# sdvx_con_AI2

## How to build the wiring in Wokwi
> can be used for demonstartion, not for actually playing
1. First place out all the components, 2 rotary encoders and 7 buttons (4 BT, 2 FX and start as seen in fig #2)
![fig #1](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_1.png)
![fig #2](https://github.com/Pugking4/sdvx_con_AI2/blob/main/sdvx_button_example.jpg)

2. Add a breadboard and connect the Arduino's 5V and GND to their associated lower rails, then which ever encoder you want to be VOL-L CLK pin to digital pin 0 (assume all pins from here to be digital), DT to pin 7, SW can be left alone, + (power) to the 5V rail on the breadboard and the GND to the GND rail on the breadboard (ignore the wiring on the top of the breadboard)
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
![20240322_185216](https://github.com/Pugking4/sdvx_con_AI2/assets/24513580/c9b0ad4c-502b-4bb1-8577-c8be6f42585d)

## Build custom PCB in KiCad
> Theoretically could be used instead of a leonardo to play USC with the current code
1. First open the KiCad Schematic Editor and add the ATmega32U4-A microcontroller using the add symbol button
<img width="1280" alt="fig #6" src="https://github.com/Pugking4/sdvx_con_AI2/assets/24513580/381e913f-de39-4018-9505-ee0254e582cf">

2. Now add all the components we'll be using which are 7x "Conn_01x02_Socket", 2x "Conn_01x04_Socket", 1x "Conn_01x06_Socket", 2x "R" and 1x "GND". Note the placing is important.
![image](https://github.com/Pugking4/sdvx_con_AI2/assets/24513580/92f539ca-ab18-4a2a-ab17-9179d0197352)

3. Now connect all the 01x02 connectors on pin 2 to the microcontroller GND pin and on pin 1 starting from J1 - j7: 26, 25, 38, 39, 40, 41 and 1, also connect the microcontroller GND pin to the common GNF symbol. Furthermore, connect the J8 01x04 connector from pin 1 - 4: 15, 14, 21 and 18. Also connect the J9 01x04 from 1 - 4 to: 15, 14, 20 and 19. Now connect the J10 from pins 1 - 6 to: 14, 3, 4, 15, to one side of R1 and then from the other side to 15 and the same for pin 6 but with R2.
![image](https://github.com/Pugking4/sdvx_con_AI2/assets/24513580/30bac342-1a78-4ee6-a09f-230ded8b55d7)

4. Click the tools dropdown at the top toolbar and select assign footprints, for the 01x02 connectors assign "Connector_JST:JST_PH_B2B-PH-SM4-TB_1x02-1MP_P2.00mm_Vertical", for the 01x04 assign "Connector_JST:JST_PH_B4B-PH-SM4-TB_1x04-1MP_P2.00mm_Vertical", for the 01x06 assign "Connector_JST:JST_PH_B6B-PH-SM4-TB_1x06-1MP_P2.00mm_Vertical", for the R assign "Resistor_SMD:R_0603_1608Metric" and lastly if KiCad has not already auto assigned the microcontroller a footprint assign it "Package_QFP:TQFP-44_10x10mm_P0.8mm".
![[Pasted image 20240323120149.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323120149.png)

5. Now open the PCB editor and select the Edge cuts layers and then click the draw rectangle tool, draw a rectangle in the dimensions of 54mm x 47mm, also select from tools in the top toolbar "Update PCB from schematic" to place all your components on the PCB try to position them to be in the middle.

![[Pasted image 20240323120639.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323120639.png)
![[Pasted image 20240323120748.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323120748.png)

6. Double click to select the B.Cu layer and then select the "Add filled zone" tool, draw a rectangle about 1mm away from the border of the edge of the PCB, when you add the copper pour make sure to select the GND net as this will guide the ratlines (the blue lines) to the GND lower plate. Also make sure to right click and select "Fill all zones" to actually fill it with copper.
![[Pasted image 20240323121155.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323121155.png)
![image](https://github.com/Pugking4/sdvx_con_AI2/assets/24513580/6fb28c19-9688-4d71-ba23-f90ed7b02820)


8. Now organise the components according to the diagram below, make sure to pay attension to the reference number (eg; J7) when placing them.
![[Pasted image 20240323121745.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323121745.png)

9. Select the "Add vias" tool from the right toolbar and place vias infront of any pin that says GND, they should be infront, not on the pin, then connect all the GND pins to the vias and for the 01x06 connect both the gnd pins to each resistor and then to the via as seen in the diagram below.
![[Pasted image 20240323122142.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323122142.png)

10. Now connect the rest of the pins to their respective microcontroller pin, if you have trouble finding routes to the pin then use my diagram as a reference. (Disregard the ratlines in the centre microcontroller)
![[Pasted image 20240323122352.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323122352.png)

Congratulations, you've created a SDVX PCB, you can view it in the 3d viewer to get a better look at it.
![[Pasted image 20240323122509.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323122509.png)
![[Pasted image 20240323122520.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323122520.png)

## Reflection

>How did your preliminary research influence and refine the initial concept of your recipe; were there any unexpected findings that led you to adjust or rethink your original idea?

When I first started researching what project to do I initially thought "What if I made a barcode scanner, but it read RGB values and not just BW?" I thought this would save space on the barcode but after doing some concept code which took me over 6 hours to complete I realised It was extremely impractical, the RGB barcode would have to be very long even for the most simple URLs
![[Pasted image 20240323124413.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323124413.png)
and with the current way I was going to implement it which was to move the scanner over a strip of colours until a black strip was reached would be very impractical. Then I thought of some hobbies I was passionate of and I thought of making a rhythm game controller, there were quite a few to choose from like a IIDX controller, Chunithm controller and if I wanted an extremely hard challenge, a maimai controller or Ongeki controller, but I ended up choosing one of the more simple ones as this was my first foray into designing a rhythm controller from scratch.

IIDX controller

![[pic003.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/pic003.png)

Chunithm controller

![[S991d3176dec447c79dd32a83287477ebh.jpg_640x640Q90.jpg_.webp]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/S991d3176dec447c79dd32a83287477ebh.jpg_640x640Q90.jpg_.webp)

maimai pico controller

![[assembled.jpg]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/assembled.jpg)

Ongeki controller

![[Photo1.jpg]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Photo1.jpg)

SDVX controller

![[9jSMV8pepo5Ch3zH2F77fpvOJUKiAGBVY0GJRbmm.jpg]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/9jSMV8pepo5Ch3zH2F77fpvOJUKiAGBVY0GJRbmm.jpg)

>What challenges did you encounter during the video scripting and design phase, and how did you adjust your approach to overcome them? Specifically, how did you ensure your video tutorial would effectively communicate the steps and details of your recipe to a diverse audience?

During the design process of the custom PCB I had numerous issues, first when I was initially designing the schematic I thought the wiring mattered so I spent quite a few hours making the connections all neat and making sure all the components were near the pins I needed on the **schematic version of the microcontroller** but to my horror after countless hours of work when I transferred my design to the PCB all the ratlines were all over the place and it was untraceable as the pin places were practically randomised compared to the actual microcontroller.
![[Pasted image 20240323130354.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323130354.png)
![[Pasted image 20240323130427.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323130427.png)

I realised after I looked at the PCB I needed to completely restart the design process, at the same time I started researching to copper pours which were filled zones completely filled with copper to act as a common ground plane or a power plane, it took quite a while to figure out how to do it but I finally figured it out and that I had to make nets in the schematic and then add a filled zone in the PCB editor. My next step was to recreate the schematic again so I I built it and made sure to keep looking at the microcontroller pin diagram to make sure all the components were lined up in the PCB and not the schematic.
![[32U4PinMapping.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/32U4PinMapping.png)
![[Pasted image 20240323131047.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323131047.png)
(this was the design at this stage but without the 01x06 connector and resistors)

I then created the PCB with relative ease as it was much easier to trace as I had figured out how to use vias to the GND plane, but I forgot one essential thing, a USB to actually connect the controller to a computer. I then had to go back to the schematic and research how to connect a USB to a microcontroller and settled on having a separate USB C PCB which are easily sourced and having a JST connector to that PCB, it wasn't too bad to add but I did have to move most components and retrace them on the PCB but then I had finally done it, with 3 reiterations of the PCB I had made the final PCB.
![[Pasted image 20240323131738.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323131738.png)
![[Pasted image 20240323131755.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323131755.png)

I also had quite a few problems when initially creating the controller IRL using the Leonardo as I could only source tiny pushbuttons which had an abnormally high weight spring in them from jaycar, and also only had a very low resolution rotary encoder to use (15 PPR vs 600 PPR which is recommended).  Finding firmware which worked too was extremely hard, first I tried making it myself but couldn't get it to work, I then attempted to find existing firmware but multiple codebases didn't work but I finally found one which worked by modifying a few lines. The 3D printing process was also extremely slow as I had to print, check if it fit and then reprint if it didn't, I probably have around half a kilo of 3d printed SDVX cases and knobs lying around my house. While I was creating the controller too I decided to go in blind as I knew the general working of the controller but that turned into a dumb idea as it lead me down a bunch of small problems which took me forever to solve.

>Based on the feedback you received during the peer review, what were the critical areas of improvement identified for your recipe and video tutorial? How did you prioritise which feedback to act upon, and what changes did you decide to implement or ignore?

Unfortunately my video/recipe wasn't ready by the time the peer review came round but I did get a friend to test it and I think it's much more realistic for them to complete it as they have practically zero background experience in mechatronics and a minor background in C coding.

>Reflecting on the entire process, how would you reimagine or further expand this project in the future? Considering all you've learned and the feedback received, design a brief proposal for a follow-up project or an advanced version of your current recipe.

In the future, especially if I knew I was making this project 2 or more months in advance then I would make a full size proper SDVX controller with a sheet metal frame and acrylic top, I'd use 600 PPR rotary encoders and a square start button from aliexpress and 60UK and samducksa 405 buttons from IST Mall which use proper microswitches, I'd also get my 2 layer PCB printed so I could use it and make the build neater.
![[Pasted image 20240323133046.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323133046.png)
![[Pasted image 20240323133100.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323133100.png)
![[Pasted image 20240323133223.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323133223.png)
![[Pasted image 20240323133329.png]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/Pasted%20image%2020240323133329.png)

My design would hopefully look and feel as premium as this controller I already own.
![[20240314_083008.jpg]](https://github.com/Pugking4/sdvx_con_AI2/blob/main/20240314_083008.jpg)
