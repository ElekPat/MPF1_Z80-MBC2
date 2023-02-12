I've always wanted to have a MicroProfessor MPF-1 but until now I didn't.
However, I own a Z80-MBC2 board from 
I explored transforming my Z80-MBC2 board as a Z80 working base (with the bonus that it will not be necessary to erase/program an EPROM each time during development) and add a keyboard and a display, the 2 elements that are missing in order to get closer to the autonomous MPF-1 solution (no need to a PC, although it can make things easier by doing cross-assembly and downloading/saving programs via the serial link offered by the Z80-MBC2 card)
I therefore studied the diagrams of the MPF-1 and Z80-MBC2 boards in order to see what was possible, with the following constraints:

- have an "identical" keyboard layout to the MPF-1
- have a display similar to the MPF-1 (4 LED digits for addresses and 2 LED digits for data)
- have STEP functionality (which uses additional logic on the MPF-1 board)
- keep all the features of the MPF-1 monitor (even if it means a lot of understanding and re-writing the code)

The Z80-MBC2 board doesn't make the Z80 bus available to the user (and that's good because I can't see myself wiring the full bus by hand with wires to the expansion board), but implements "virtual" peripherals on the Z80 bus via the ATmega (peripherals which can be SPI, like the SD card, or I2C like the GPIO expander).

So I undertook the construction of an auxiliary board (on a prototype empty PCB to go faster than a custom PCB) on which the 36 keys of the keyboard appear, a PIC16F1619 (others may be suitable, it is what I had on hand) which will manage this keyboard (6x6 matrix but only 32 keys are managed, as on the original MPF-1, the other 4 keys, those in the left column, go to the Z80 directly or through the ATmega), and the Buzzer and will communicate via an I2C link with the ATmega, 3 leds (the 2 original leds of the MPF-1 + 1 blue led controlled by the PIC16 embedded FW).
For the screen, I chose a standard 2-line, 16-character LCD module (I didn't have a simple I2C 7-segment LED display solution at hand) with a small I2C I/O expander card (PCF8574) so that it is also seen as a peripheral of the Z80 through the I2C bus of the ATmega.
The advantage is that the hexadecimal characters are easily readable (since I use the "real" letters A to F)... as well as the names of the registers...
I only use (for the moment) the first 6 characters of each line to stick as close as possible to the MPF-1 configuration (the bottom line is used to display the "." which are missing on the digits).

I wrote the keyboard and Buzzer management C project for the PIC16 (20 pins/17 I/Os) with MPLAB X and the XC8 compiler. There isn't a free pin, I even re-use the ICSP pins for the I2C connection... we would probably be more comfortable with a 28 pin device?
Using a Pickit Serial (and its associated software on the PC), I validated that an external I2C host could indeed access the control/read registers of the keyboard, the Buzzer and the LED.

And I made the following additions/modifications on the Z80-MBC2 board:
- transition from ATmega32A to ATmega644A (to have more program memory in order to put the MPF-1 monitor and all the new routines to manage the display and the external keyboard, and also because that's what I had to available (by removing the Forth and/or the BASIC from the Z80-MBC2, everything should fit on an ATmega32A)
- modification of the reset between the ATmega and the Z80 so that the external RS button of the MPF-1 acts on the reset of the Z80 (and not that of the ATmega)
- jumper SJ1 bridged to have the M1 signal of the Z80 on the ATmega (for the STEP functionality) => INT0 triggering of the ATmega
- MONI button connected to the MCU_RTS pin of the ATmega (pin not used in the original IOS_Lite software)
- connection of the USER signal of the ATmega on the NMI pin of the Z80 (for STEP and MONI functionalities)
- connection of the external INT button on the Z80 (not yet tested)
- modification of the IOS_Lite sketch in order to manage the display and the keyboard

After validating that the display, the keyboard and the buzzer are controllable by the ATmega, I took care of adding the virtual peripherals of the Z80 (IN and OUT commands intercepted by the ATmega) in the IOS_Lite code so that the Z80 has also access the display, the keyboard and the buzzer.
Once all this was in place, I wrote a small Z80 assembler program to validate that everything was working correctly.

Now it was time to tackle the modifications of the MPF-1 monitor: and there it was necessary first to understand how it all worked, in particular the stepping mechanism (STEP)!
This is what caused me the most problems, in particular since I am not used to development on Atmel, and I did not manage to optimize the ISR routine (necessary to count the machine cycles M1 of the Z80), which means that for the moment it only works for a clock of 4 MHz on the Z80 (the possibility of 8 MHz of the Z80-MBC2 is therefore not compatible with the MPF-1 mode).

I replaced TapeWR and TapeRD with SAVE and LOAD (well that's what I call them):
- SAVE which sends the contents of a memory block in Intel HEX format on the serial link of the ATmega (so can be saved on the PC with for example Teraterm in "Log" mode), for this I wrote a Z80 assembler routine dedicated but which uses the parameter input system (start address and end address of the memory block) of the MPF-1 monitor
- LOAD which loads an Intel HEX file from the ATmega serial link (sent for example by the PC with Teraterm "Send File"), for this I directly use the iLoad routine of IOS_Lite
