## L6474
Stepper motor driver library for Arduino Uno R4 using the X-NUCLEO-IHM01A1 shield.

### 1. Introduction
----------------
This library allows an Arduino Uno R4 to control one or more stepper motors using the X-NUCLEO-IHM01A1 shield. It supports daisy-chaining multiple shields.

### 2. Installing the library into the Arduino IDE
------------------------------------------------
This library can be installed using the Arduino Library Manager. Open the IDE, go to `Sketch > Include Library > Manage Libraries...`, and search for "L6474-stepper-motor-driver".
  
### 3. Library content
---------------------
The library contains the following files:
- `l6474.cpp`: core functions of the L6474 class
- `l6474.h`: declaration of the L6474 class and the associated definitions
- `l6474_target_config.h`: predefines values for the L6474 registers and the shields parameters (speed profile)
- `keywords.txt`: definitions of the L6474 class keywords for the syntax highlighting with the Arduino IDE.

### 4. Example Sketches
--------------------
Example sketches are provided to demonstrate driving one or more stepper motors and accessing the L6474 registers.

### 5. Library features
--------------------
The L6474 library has the following features: 
- L6474 registers read, write 
- Uno R4 and X-NUCLEO-IHM01A1 shield configuration (GPIOs, Timers, IRQs)
- Speed profile configuration 
- Motion commands 
- FLAG interrupts handling (alarms reporting)
- Microstepping handling
- Daisy chaining to handle up to three X-NUCLEO-IHM01A shields

Depending of the shields number, the library will:
- setup the required GPIOs to handle the motor directions and the FLAG interrupt
- initialize the hardware timers that act as the step clock frequency generators
- initialize the speed profile (acceleration, deceleration, min and max speed) of each shield by using the parameters of the file `l6474_target_config.h`
- starts the SPI library to communicate with the L6474 chips
- Release the reset of each of the L6474 chips
- Disable the power bridge and clear the status flags of the L6474 chips
- load the registers of each L6474 with the predefined values from `l6474_target_config.h`

Once the initialization is done, the user can modify the L6474 registers and speed profile configurations as desired. Most of the functions of the library take a shield Id (from 0 to 2) as input parameter. It gives the user the possibility to specify which of the three shields configuration they want to modify. 

The user can also write a callback function and attach it to the Flag interrupt handler depending on the actions they want to perform when an alarm is reported (read the flags, clear and read the flags...)

Then, they can request the move of one or several motors (still using the same principle of shield Id). This request can be:
- to move for a given number of steps in a specified direction
- to go to a specific position 
- to run until reception of a new instruction

On reception of this request, the library will enable a hardware timer to trigger a software interrupt (ISR). At each pulse period, the ISR physically executes the step pulse (`digitalWrite`) and updates the microcontroller's step count. The motor starts moving by using the minimum speed parameter. At each step, the speed is increased using the acceleration parameter.

If the target position is far enough, the motor will perform a trapezoidal move:
- accelerating phase using the shield acceleration parameter
- steady phase where the motor turns at max speed
- decelerating phase using the shield deceleration parameter
- stop at the targeted position

Otherwise, if the target position does not allow reaching the max speed, the motor will perform a triangular move:
- accelerating phase using the shield acceleration parameter
- decelerating phase using the shield deceleration parameter
- stop at the targeted position

A moving command can be stopped at any moment:
- either by a soft stop which progressively decreases the speed using the deceleration parameter. Once the minimum speed is reached, the motor is stopped.
- or by an hard stop command which immediately stops the motor.
In both cases, once the motor is stopped, the power bridge is automatically disabled.

To avoid sending a new command to a shield before the completion of the previous one, the library offers a `WaitWhileActive()` command which locks the program execution till the motor ends moving.

The library also offers the possibility to change the step mode (from full step till 1/16 microstep mode) for a given shield. When the step mode is change, the current position (`ABS_POSITION` register) is automatically reset but it is up to the user to update the speed profile (max and min speed, acceleration deceleration).

### 6. Architectural Design & Motion Control
----------------------------------------
This library is explicitly designed for high-precision motion control, where positional drift over long periods is unacceptable. To achieve this, the architecture utilizes a "Semi-Smart" distributed approach:

* **Hardware Frequency, Software Execution (Preventing Over-Stepping)**
  The Uno R4's ARM FSP hardware timers are used strictly to calculate the frequency ensuring zero drift during acceleration curves. However, the physical step pulse is executed manually in software via an Interrupt Service Routine (ISR). This prevents the "Over-Stepping" problem inherent to pure hardware PWM by a microcontroller, where the hardware timer fires extra pulses while the CPU is busy trying to shut the timer off. By toggling the pin in software, the CPU guarantees the motor stops accurately.
  
* **Hybrid Closed-Loop Counting ("Trust, but Verify")**
  The CPU tracks the motor's position in a fast software variable (`relativePos`) to perform rapid velocity math without waiting for slow SPI communications. However, every 16 steps, the CPU pauses and queries the L6474's internal hardware `ABS_POS` register via SPI. The L6474 acts as the physical auditor. If electromagnetic interference (EMI) or a micro-fault causes a pulse to be missed or duplicated on the physical wire, the software instantly corrects its math to match the hardware's reality, absorbing noise without corrupting coordinates.

* **Strict Interrupt Priority**
  Because pulse execution relies on software ISRs, the stepper motor timers are hardcoded to **Priority 1** (the highest priority). In multi-sensor systems, this ensures the step generation permanently outranks other peripherals (like I2C displays or Quadrature Encoders), preventing microsecond jitter caused by interrupt collisions.

### 7. Library required resources
------------------------------
The L6474 library uses the standard Arduino SPI library. The step clock for each shield is generated by a hardware timer on the Uno R4.
<br>
Pinout for a single shield:
<br>

| **Function** | **Digital Pin** | **Notes** |
| :--- | :--- | :--- |
| Flag Interrupt | 2 | Shared by all shields |
| L6474 Reset | 8 | Shared by all shields |
| SPI SS | 10 | SPI Slave Select |
| SPI MOSI | 11 | SPI Master Out Slave In |
| SPI MISO | 12 | SPI Master In Slave Out |
| SPI SCK | 13 | SPI Serial Clock |
| Step Clock (PWM) | 9 | Shield 0 |
| Direction | 7 | Shield 0 |

<br>

### 8. Boards wiring for multi-motors configurations
------------------------------------------------
If you want to drive several motors with the library, you will first need to have 2 or 3 X-NUCLEO-IHM01A1 boards.
The driving of the different motors is done thanks to daisy chaining (see next paragraph). To handle this daising chaining, the boards have two crossover pins that are combined with some shunt (0K) resistors to propagate the SDO (or MOSI) of one board to the SDI (or MISO) of the following one.

By default, the shunt resistors board are set for 1 board configuration (R1, R4, R7, R12 resistors mounted).

For 2 boards configuration:
- the first board you have to move these resistors to have R1, R4, R7, R10 resistors mounted
- the second must have R2, R5, R8, R12 resistors mounted

For 3 boards configuration:
- the first must have R1, R4, R7, R10 resistors mounted
- the second must have R2, R5, R8, R11 resistors mounted
- the third must have R3, R6, R9, R12 resistors mounted

To have more details you can have a look in paragraph 2.2 of ST document " UM1857: Stepper motor driver expansion board based on L6474 " that you can find here: http://www.st.com/st-web-ui/static/active/en/resource/technical/document/user_manual/DM00156746.pdf

Once the shunt resistors are correctly mounted, you only have to plug on board on top of the other to have a 2 or 3 boards configuration.

Resistors R25 and R24 could also be moved to use another CS (chip select) line or Clock line. But in the case of this library, you can keep the default configuration.

### 9. Additionnal information regarding daisy chaining
----------------------------------------------------
The daisy chaining is used to send command via the SPI from the Uno to several X-NUCLEO-IHM01A1 boards.
The purpose of these commands can be to enable/disable the bridges, set the parameters of the L6474 (to dynamically adjust the torques for examples), to know the position, to get the alarm status (over current detection, over temperature,...) of the L6474.
<br>
The principles of the daisy chaining are the following:
- all boards share the same clock and chip select.
- the SDO of the Uno is linked to the SDI of the first shield
- then each SDO of one shield is linked to the SDI of the following shield (thanks to the two additional crossover pins)
- the SDO of the last shieled is linked to the SDI of the UNO
- the data are transmitted byte by byte.
- When the CS is high, the SPI works as a delay line: at each clock enabling , a byte is read from the SDI and pushed to the SDO
- When the CS is released, the last byte received by the L6474 is latched and decoded. The answer is prepared to be sent out vai SDO at the next clock enabling.

For full details, see ST's application note: "AN4290: L647x, L648x and powerSTEP01 family communication protocol" http://www.st.com/st-web-ui/static/active/en/resource/technical/document/application_note/DM00082126.pdf
and specifically at figure 9 where there is a time diagram with the clock and several devices!

From the point of view of a library user, the daisy chaining is hidden, and its use is quite simple as you only have to specify the index (from 0 to 2) of the targetted board to use it!

For example, if you want to get the status of the first board you have to write:<br>
`uint16_t statusRegister = myL6474.CmdGetStatus(0);` <br>
to get the status of the second shield:<br>
`uint16_t statusRegister = myL6474.CmdGetStatus(1);` <br>
to get the status of the third shield:<br>
`uint16_t statusRegister = myL6474.CmdGetStatus(2);` <br>

To set the torque regulation current of the first shield board to 625 mA (caution only multiple of 31.25 are supported):<br>
`myL6474.CmdSetParam(0, L6474_TVAL, 625);` <br>

To set the torque regulation current of the second shield board to 1000 mA<br>
`myL6474.CmdSetParam(2, L6474_TVAL, 1000);` <br>

### 10. Drive one motor by uart commands
------------------------------------
If you want to drive one motor by sending commands from a PC to the uno board by Uart, you can take the sketch "L6474SketchFor1MotorShieldDrivenByUart".

To use it:
- You have to install the L6474 library 
- The Uno must have a X-Nucleo-Ihm01A1 on top of it. A stepper motor has to be connected to the bridges of the expansion board and its external power supply must be enabled.
- The terminal port must be configured to 9600 baud, new line handled with option “CR+LF” and I recommend to enable local echo.
- Then, just download the sketch to the Arduino Uno with the IDE.   

If your setup is correct, as this step you should see the token :”>>:” at the serial terminal.
It means that the Uno firmware is expecting some instructions.
Then you have to enter one of the following command line:

- `C1`                -> to get the acceleration
- `C2`                -> to get the current speed
- `C3`                -> to Get the deceleration
- `C4`                -> to Get the shield state
- `C5`                -> to Get the shield the FW version
- `C6`                -> to Get the Mark
- `C7`                -> to Get the max speed state
- `C8`                -> to Get the min speed shield state
- `C9`                -> to Get the current position
- `C10`               -> to go home the shield state
- `C11`               -> to go to mark the position
- `C12 <arg1>`        -> to go to `<arg1>` position
- `C13`               -> to make a hard stop
- `C14 <arg1> <arg2>` -> to move of `<arg2>` steps in forward direction if `<arg1>` is 0, in backward direction if `<arg1>` is 1
- `C15`               -> to reset all shields
- `C16 <arg1>`        -> to run in forward direction if `<arg1>` is 0, in backward direction if `<arg1>` is 1
- `C17 <arg1>`        -> to set the acceleration to `<arg1>`
- `C18 <arg1>`        -> to set the deceleration to `<arg1>`
- `C19`               -> to set current position to be the home position
- `C20`               -> to set current position to be the mark position
- `C21 <arg1>`        -> to set max speed to `<arg1>`
- `C22 <arg1>`        -> to set min speed to `<arg1>`
- `C23`               -> to make a soft stop
- `C24`               -> to wait while the motor is active
- `C25`               -> to send a disable command (bridges off)
- `C26`               -> to send a enable command (bridges on)
- `C27 <arg1>`        -> to send a get param command with `<arg1>` as register address
- `C28`               -> to send a get status command (read and clear status register)
- `C29`               -> to send a nop command 
- `C30 <arg1> <arg2>` -> to send a set param command with `<arg1>` as register address, `<arg2>` as value 
- `C31`               -> to read status register (no clear is done)
- `C32`               -> to reset the shield
- `C33`               -> to release the reset
- `C34 <arg1>`        -> to set the step mode to full step if `<arg1>` is 1, to half step if `<arg1>` is 2, to 1/4 step if `<arg1>` is 4, to 1/8 step if `<arg1>` is 8, to 1/16 step if `<arg1>` is 16
- `C35 <arg1>`        -> to set direction: forward direction if `<arg1>` is 0, backward direction if `<arg1>` is 1

### 11. Motivation & Support
------------------------------------
This port of the L6474 library to the Arduino Uno R4 was originally developed to support high-precision, closed-loop motion control for other projects of mine. For a deeper dive into its use, please visit my [website](https://mattykakesmakes.com/).

If you enjoyed using this library, and want to support my contributions, please feel free to [Buy me a Coffee](https://buymeacoffee.com/mattykakesmakes) 🍵😉