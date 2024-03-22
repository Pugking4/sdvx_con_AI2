# sdvx_con_AI2

## How to build the wiring in Wokwi
> can be used for demonstartion, not for actually playing
1. First place out all the components, 2 rotary encoders and 7 buttons (4 BT, 2 FX and start as seen in fig #2)
![fig #1](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_1.png)
![fig #2](https://github.com/Pugking4/sdvx_con_AI2/blob/main/sdvx_button_example.jpg)

2. Add a breadboard and connect the ardiuno's 5V and GND to their associated lower rails, then which ever encoder you want to be VOL-L CLK pin to digital pin 0 (assume all pins from here to be digital), DT to pin 7, SW can be left alone, + (power) to the 5V rail on the breadboard and the GND to the GND rail on the breadboard (ignore the wiring on the top of the breadboard)
![fig #1](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_2.png)

3. Repeat for VOL-R encoder with the pin assignments of CLK to 3, DT to 5, + (power) to bb (breadboard) power rail and GND to bb GND rail.
![fig #1](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_3.png)

4. Place all the buttons on the bb in a line and then attach the 2nd ardiuno GND pin to the upper GND bb rail.
![fig #1](https://github.com/Pugking4/sdvx_con_AI2/blob/main/wokwi_4.png)


