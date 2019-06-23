# ECE362_project
My academic project for Microprocessor Systems and Interfacing course

# Final Project: Crops Plantation System Simulator

Michael Miller, Roger Biak

Indiana University-Purdue University Indianapolis

ECE 362 - Fall 2017

Department of Electrical and Computer Engineering

December 6, 2017

## Introduction

### Purpose:

The purpose of the final project is to test the comprehensive understanding of 68HC12 Microprocessor Systems and Interfacing. Using crops plantation as a topic, everything that is covered in the labs is combined to make a single system. This project covers the topics Arithmetic Instructions, Branch Instructions, Subroutines, Symbol Definitions and References, Logic and Bit Instructions, Hex Keypad,  Real Time Interrupts, Pulse Width Modulation, and Passing Values Through the Stack. The final result of the project should demonstrate a good understanding all the mentioned topics. This report will go into further details and explanations and how each functionality is constructed and how the are integrated in system as a whole.

### Assumptions Made:

 The project's requirement for extra functionality was the only element that inspired certain assumptions to be made--mainly how any additional features would interact or be in conflict with required the functions. This extra feature, was to give the user the option when to harvest crops, while the original specifications requires a message to prompt the user the moment crops became harvestable. This option would allow the implementation of extra plant stages, overall making the system more complex. Since this would make the system more complex it was assumed that it would okay to negotiate with the TA to accept this change.

### Design

Peripherals (Refer to figure 1)

1. **LCD Screen** is used for the system screen. After showing the welcome screen for five seconds after the program starts, it navigates the user.

2. **Hexpad** Keypad serves as the controller the system.

3. **IRQ:** Pests is detected when IRQ is pressed. The user is forced to spray pesticide when bugs are detected.

4. **Push Button (PB)** serves as five different roles. 
	+ PB serves as a _next_ button in the tutorial.
	+ PB serves as a _ready_ button in the calibration.
	+ PB is pressed when finishing spraying pesticides.
	+ PB is used to adjust the duty cycle rate for fertilizing and watering the field.
	+ PB is used to check the crops stage.

5. **Switches**
	+ **Switch 1 (bit 0)**: If and only if Switch 1 is turned on, the LCD screen shows the soil fertility and the fertilizer spreading rate. The user as an option to adjust the duty cycle rate using PB. Switch 1 must be turned off after adjusting the watering duty cycle.
	+ **Switch 2 (bit 1):** If and only if Switch 2 is turned on, the LCD screen shows the humidity of the field and the watering duty cycle rate. The user as an option to adjust the duty cycle rate (sprinkling water rate) using PB. Switch 2 must be turned off after adjusting the watering duty cycle.
	+ **Switch 4 (bit 3):** Must be turned on for the stepper motor to work.
	+ **Switch 8 (bit 7):** If and only if Switch 8 is turned on, the user is directed to calibrate the POT. When POT is turned on to the max, the user is directed to press PB. The user cannot get out of the calibration screen as long as Switch 8 is turned on.

6. **LEDs** are used throughout the whole system. Whenever there is an action in the system, the LEDs are lit. Each action in the system has a different (pattern) lightning style.

7. **Potentiometer** is used to spray pesticide. Turning it on to the max means the whole field is sprayed.

8. **AMP** controls sound volume.

9. **Speaker** replicates the sound system of the program. It plays different tunes (songs) for different action.

10.  **Stepper Motor** replicate 3 agriculture machinery. It spins counter-clockwise  in fast speed when plowing, clockwise in relatively slow speed when seeding, and counter-clockwise relatively slow when harvesting the corps.

11. **DC Motor** spins when sprinkler system or fertilizer spreader is activated. The speed of the motor can be adjusted by changing the duty cycle.

### Software Implementation

1. **Real Time Interrupt (RTI)**

This was the most import routines because it was used in every other routine. The main concept in our implementation was having several different timing variables that would be automatically incremented at specified rates. Since the RTI was configure to be called every 1 millisecond the shortest and first timing variable to be increment was a 1 millisecond time variable `delay1ms`, `d1ms`. It was arbitrarily chosen to have the additional time variables as multiples of ten `delay10ms`, `delay100ms`, `delay1s`, `d10ms`, `d100ms`, `d1000ms`. To increment `delay10ms` the program compares `d1ms` to ten. If `d1ms` was lower than 10 the program would branch out to `END_RTI`, if `d1ms` was equal to 10 we would reset `d1ms` and increment `delay10ms`. Similarly, to increment `delay100ms` the program would compare `d10ms` to ten. Implement the same procedure. To understand why each timing variable needed a twin e.g. `d10ms`, `delay10ms` was so that one variable `delay10ms` could be incremented to 255 before resetting to 0, which was necessary for certain subroutines that required more than 10 timing variable increments.

Additionally, one project requirement was to allow the plant to age into different growing stages based on the RTI. The program uses the timing variable `d1000ms` to keep track to the crops age in seconds. The `END_RTI` procedure would always check `d1000ms` by comparing it specific values to determine when to update growing stage variable `PlantStage` and field status LCD data `status_disp`. For visual understanding see ***Figure 9: RTI Chart***.

2. **Interrupt Request (IRQ)**

The implementation required the subroutine to enable Interrupts from within the RTI because  the program needed a timing variable. Since this IRQ could be called anyway in the system the subroutine had to store timing variables to the stack, and set a flag variable `IRQ_ON`, in order prevent the plants from being aged by the RTI while the program was idling in the IRQ the subroutine also needed to prevent the IRQ from being called within it self. The was achieved with `bclr   INTCR, #$C0` to disabled IRQ. After the main conducting the main functionality the IRQ was re-enable it at the end of the subroutine. For visual understanding see ***Figure 5: IRQ Flow Chart***.

3. **Hexpad/DIP Switches/Push Button**

The hexpad routine uses post-auto-increment addressing using a 4 element array to scan the rows. After a button press is detected and debounced (ensures stable signal) a table lookup routine is called to convert the raw input to the printed symbol on the board. After button is release we pass the detected key press `mem_u` with `SendsChr` to play a corresponding tone and return to from subroutine.

In the event a button press is not detected after all rows are scanned. The subroutine checks the DIP Switches `1[CheckSoil]` , `2[CheckWater]`, `8[Calibrate]`. If any of the specified DIP switch is exclusively on, the corresponding routine will be called then the Hexpad returns from subroutine. If no valid DIP switch configuration is detected the program it will check if the Push Button is pressed. If the Push Button is pressed it will call the Calibrate routine then hexpad will return from subroutine. If nothing is detected the hexpad with return from subroutine.

4. **Sprinkle Water/Spread Fertilizer (DC Motor)**

The Sprinkle Water and Spreader fertilizing are very similar as they both use Pulse Width Modulation (PWM) control of the DC motor. The total PWM cycle period (on period + off period) is 100 milliseconds (when delay10ms = 10). The routine use two timing variables (1 second) for controlling how long the entire Routine runs, and `10ms` for the actual PWM control. The subroutine first checks if (1 second) is greater than (5) if not then compares if `10ms` is less than or equal to `t_on` to turn DC motor. If `10ms` is greater than `t_on` and less than (total cycle period) it will turn of DC. Once `10ms` increments past the (total Cycle period) it reset `10ms` and branches branch restarts the PWM procedure again, till (1 second) is equal to 5)

5. **Speaker/LEDs**

The speaker and LEDs had to simplest implementation due to the structure of the RTI. The main concept was to use a timing variable from the RTI to load the LEDs and Speaker with a value that would automatically increment at a predetermined rate. This means that the loop did not need to increment anything at all to allow the program to exit the loop at some point. For example the intro song uses the timing variable `delay10ms` to play the values from 0 to 125. This means that the entire song last slightly more than 1250 milliseconds`10ms\*125=1250ms`. And each tone is played for 10 milliseconds. An important detail to understand this implementation is that although it might appear the program loops only 125 times is actually many time that much. This is due the understanding the program instructions in the loop are performed in the microsecond range. So the programing is looping several times with the same unchanged value`delay10ms` being stored the LEDs Speaker till the RTI has incremented the value `delay10ms`. This functionality is was allows the speaker to hold a tone.

6. **Plowing/Seeding Planting (Stepper Motor)**

The Field Plowing and Seed Planting Mode became rather more complex than necessary but it help exceed the requirements. The implementation has the LCD show symbols changes from left to right in the top row then right to left in the bottom row with each row having two spaces in the middle. For each symbol change the stepper motor will cycle through the 4 element array, and then the board will play a sound. The subroutine uses register B loaded with the value 32 to coordinate LCD functions and a 32 element array `status_disp` to store LCD data about the field. For the first row LCD can show new symbols till register B decrements to 16. Register B is also checked(compared) for the values 25 and 24 to determine when to write a space to the LCD.  Keep in mind the stepper motor is loaded with 4 elements from an array to slightly rotate it after each character is written to the LCD. For the Bottom row register Y that is being use to point to the 32 element array `status_disp` is modified with register B to point that last element (index 31). Similarly the program will check register B for the values 8 and 9 to know when to write and display a space instead of a new character symbol. Once B decrements to zero the program has successfully printed all symbols, while turning the stepper motor, and playing a noise for each 32 character space available on the LCD.

### Design Changes

The only design that we modified is the harvest system. The project requires the system to show a warning message when the crops is ready and force the user to harvest. To give the user more feeling of real life plantation, we created a harvest button. According to our new modification, the user can press B from the Hexpad to harvest at anytime except when the crops is in the seed stage. Our idea is that the user can harvest at any desired time, however, there won't be any crops yield if it is not harvested in the ready stage. If the user doesn't harvest in time after the ready stage (we gave them 30 seconds), the crops will be rotten.

### Addition to the Projects

1. **Tutorial** was created to guide the user how the system works. The user has a choice to go through the tutorial or directly to the main menu. This choice is given right after the welcome screen disappear.
2. **Calibration:** Since the project requires the max value of potentiometer to be equals to 100% and min value to be 0%, it is necessary to know what the max value is in order to get percentage. Calibration is was created to store the max value of the potentiometer into some variable.
3. **Extra stages** sprout and rotten, were added in the growth of the crops. Sprout comes twenty seconds after the seeds are planted and the crops become rotten if the user doesn't harvest the crops thirty seconds after it is ready to be harvested.
4. **Harvest Option** is given to the users when C (Hexpad) is pressed. The user can harvest when the crops is in growing, matured, ready to be harvest, and rotten stages. However, the user does not get anything if the crops is not harvested in ready stage.
5. **Return to Main Menu** button (F in Hexpad) is created for the user's convenience.

### Working Style

The whole project was done together as a team for the following reason

1. to avoid the difficulties of understanding different styles of coding.
2. to avoid the confusions of having different variables and subroutine names
3. it is necessary to check the code again and again to see if it works or not. 

This style was very effective when either team member was having difficulty debugging code. It also made it easier for team members to start from where the other left off, especially when taking breaks to recover from fatigue.

### Functionality of Project

 It was a success. The whole project requirements are met and there is no bugs in the program. The system functions perfectly.

### Discussion and Suggestions

The entire project exercised all the topics learned throughout the course in the Lab. The most important detail in completing the assignment was how we responded to technical difficulties and the development process.  Our first issue encountered was trying to get the speaker to function properly. After spending too long on the section with no progress we decided skip it and worked on the LCD and menu navigation. This decision set the theme for focusing on more generalized functionalities and avoid becoming too involved in small details. After getting the menu navigation to function properly we needed improve the RTI to include additional timing variables. Once the DC motor and Stepper Motor was implemented properly, we began to structure our Plant stage code. The Plant stage section went through several versions till it was implemented properly, due to the complexity of involving it with the RTI timing.

 While the entire project successfully fulfilled all the requirements and additional features there were still a couple planed features to mention. One of the features that was devised but was not fully implemented was to introduce at crop quality variable that would be calculated in harvest mode based on data about watering, fertilizing, pesticides, etc. The feature could then be expanded on by introducing even more metrics to affect crop quality like herbicide, climate, and diseases.

### Conclusion

 The whole project building experience was a very useful learning exercise that help incorporate all the labs together into one thing. As mentioned before the project has been extensively tested but has yet to show any signs of errors or bugs. While not every line of code was commented, each subroutine section has been provided with a sufficient amount of comments to explain the main process being executed. Due to timing constraints and changes in implementation there may be several redundancies, unused variables and subroutines. Although the process had a some rough moments due to changes in implementation it still worked out for the better.

### Appendixes I
---
#### User Manual

1. **Getting started** : A welcome message is displayed for five seconds at the beginning and the user has an option to choose Tutorial or Main Menu.

1. **Calibration:** Turn on Switch 8 to do a calibration. This must be done in the Main Menu.

1. **Return to Main Menu:** Click F in the Hexpad.

1. **Spray Pesticides:** Click IRQ to spray.

1. **Grow Crops:** The field must be plowed before planting the seeds and all the field must be empty (harvested) before plowing. The user can grow crops when these two conditions are met.

1. **Field Plowing and Seed Planting:** From the Main Menu, choose the "New Farming Cycle" option. If the farm has crops, the program will not let the user to go to the "New Farming Cycle". The crops must be harvested before hand. Refer to _Figure 2: Main Menu Flow Chart._

1. **Watering and Fertilizing the field:** From the Main Menu, choose &#39;Fertilizing&#39; option. The watering and fertilizing speed can be adjusted as directed in _Checking Field Condition._ Refer to _Figure 2: Main Menu Flow Chart._

1. **Harvest:** Click B in the Hexpad to harvest.

1. **Checking Field Condition:**
	+ Turn on Switch 1 to check the fertility of the soil. Press PB to adjusts fertilizer spreading rate. Turn off Switch 1 to return.
	+ Turn on Switch 2 to check the humidity of the field. Press PB to adjusts water sprinkling rate. Turn off Switch 2 to return.

1. **Checking Crops Stage:** Click PB to check the crops stage. It is suggested to harvest when the crops is in Read to harvest stage.

### Appendixes II
---
#### File Descriptions

Calibrated.asm - Contains `Calibrated` subroutinw

CheckCropsCond.asm - Contains `CheckCropsCond` (Depreciated) subroutine

Delay.asm - Contains `Delay1` subroutine

Fertilize.asm - Contains `Fertilize` subroutine 

FertTest.asm - Contains `FertTest` subroutine

GrowingStage.asm - Contains `GrowingStage` subroutine

Harvest.asm - Contains `Harvest` subroutine

HarvestStage.asm - Contains `HarvestStage` subroutine

MainMenu.asm - Contains `MainsMenu` (Depreciated) subroutine

MaturedStage - Contains `MaturedStage` subroutine

NewFarmCyc.asm - Contains `NewFarmCyc` subroutine

NoCropsStage.asm - Contains `NoCropsStage` subroutine

PestDetected.asm - Contains `PestDetected` (Depreciated) subroutine

PestGuide.asm - Contains `PestGuide` subroutine

Pesticides.asm - Contains `Pesticides` subroutine

ReadyStage.asm - Contains `ReadyStage` subroutine

RottenStage.asm - Contains `RottenStage` subroutine

RTIDebounce.asm - Contains `RTIDebounce` subroutine

RTIDelay.asm - Contains `RTIDelay` subroutine

SeedStage.asm - Contains `SeedStage` subroutine

SetMax.asm - Contains `SetMax` subroutine

SpreadingFert.asm - Contains `SpreadingFert`subroutine

SprinklingH2o - Contains `SprinklingH2o` subroutine

SproutStage.asm - Contains `SproutStage` subroutine

TableSearch.asm - Contains `Search` subroutine

TransferDisp.asm - Contains `TransferDisp` (Depreciated) subroutine

UpdateStatus.asm - Contains `UpdateStatus` subroutine

Warning.asm - Contains `Warning` subroutine

Warning2.asm - Contains `Warning2` subroutine

WaterTest.asm - Contains `WaterTest` subroutine

Welcome.asm - Contains `Welcome` (Depreciated) subroutine

main.asm – Contains main code

### System Modules and Subroutines
***Ordered by code structure in main.as***

`Hexpad` - Module that scans for inputs on the Hexpad, DIP Switches, and Push Button.

`Search` - Table lookup to convert raw hexpad data into characters label on board.

`Calibrate` - Module to perform calibration of the potentiometer.

`SetMax` - Display message telling the user to set the potentiometer all the way up.

`Calibrated` - Displays message telling the user has successfully calibrated the potentiometer and how to return back to menu screen.

`CheckSoil` -  Module used to perform Check Soil Fertilization Function and PWM DC motor control use by Spread Fertilizer feature.

`FertTest` -  Displays template for Check Soil Fertilization function to LCD.

`CheckWater` - Module use to perform Check Soil Humidity Function and PWM DC motor control use by Sprinkle Watering Feature

`WaterTest` -  Displays template for Check Soil Humidity function to LCD.

`HarvestCrops` - Module use to perform harvest feature

`FieldPlowing` - Module use to perform Field Plowing functions

`SeedPlanting` - Module used to perform Seed Planting functions.

`SpreadFertilizer` -  Module use to perform Spread Fertilizer feature

`SpreadingFert` -  Displays Message when system is Spreading Fertilizer.

`SprinkleWater` - Module use to perform Watering feature

`SpreadingH2O` - Displays Message when system is Sprinkling Water

`CheckCrops1` - Module use to display the Plant Stage and Age of the crops and then displays the field, showing unique
characters/symbols for each different plant stage.

`SeedStage` - Displays “Seed” for Plant Stage in Check Crops Function

`SproutStage` - Displays “Sprout” for Plant Stage in Check Crops Function 

`GrowingStage` - Displays “Growing” for Plant Stage in Check Crops Function 

`ReadyStage` - Displays “Ready” for Plant Stage in Check Crops Function 

`HarvestedStage` - Displays “Harvested” for Plant Stage in Check Crops Function 

`NoCropsStage` - Displays messages that there are no crops planted yet. 

`RTI_ISR` - Module use to perform Real-time growing of the crops

`RTI_IRQ` - Module use to perform Spray Pesticide Functions

`PestGuide` - Displays messages to spray pesticides.

`Potentiometer` - Module use to perform Spray Pesticides functions

`Pesticides` - Displays template for Spray Pesticide Function

`read_pot` - Store a reading from the potentiometer and stores it in the D register

`UpdateStatus` - Use to overwrite 32 elements array for (status_disp) with a single character with is passed through the Y index Register

`display_string` -  Use to write to LCD screen, an address for a 32 element array (which contains the characters you want on the LCD) passed through the D Register.

`SendsChar` - Use to set the frequency the speaker will play a tone at when PlayTone is called. The 8 bit parameter is push the stack, then calling SendsChar will use that value to set the frequency.

`PlayTone` - Use to play tone on the speaker.

`Delay1` - Software delay loop (100 ms).

`RTIDebounce` - RTI delay (100ms)

`Harvest` -  Prompts users on LCD with Yes/No Harvest Option

`Fertilize` - Displays Fertilizing Option Menu on the LCD

`NewFarmingCyc` - Displays New Framing Cycle Menu

`Warning` - Displays Error if user tries to enter New Farming Cycling if there are crops still on the field.

`Warning2` -  Displays Error message if user tries to Plant Seed before plowing Field.

`init_LCD` - Use to initialize the LCD

## Figures 

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_1.jpg)

***Figure 1: HC12BRD4 SMW***

1. LCD Screen
2. Hexpad
3. IRQ
4. Push Button
5. Switches
6. LEDs
7. Potentiometer
8. AMP (controls sound volume)
9. Speaker
10. Stepper Motor
11. DC Motor

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_2.jpg)

***Figure 2: Main Menu Flow Chart***

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_3.jpg)

***Figure 3: Switch 1 and Switch 2 Flow Charts***

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_4.jpg)

***Figure 4: Calibration Flow Chart***

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_5.jpg)

***Figure 5: IRQ Flow Chart***

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_6.jpg)

***Figure 6: Field Action LCD***

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_7.jpg)

***Figure 7: Stepper Motor Chart***

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_8.jpg)

***Figure 8: Fertilizing Chart***

![images](https://github.com/digitalhabitat/ECE362_project/blob/master/images/figure_9.jpg)

***Figure 9: RTI Chart***
