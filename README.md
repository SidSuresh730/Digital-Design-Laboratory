# Color-changing-text-editor
## Abstract
This project was a culmination of the various uses for an FPGA (Nexys A7-100T) learned in the ECE 4401 course. 
Specifically, it used PS/2 protocol, VGA protocol, the on-board switches, and the AXI4-Lite bus interface. 
The digital system constructed in this class read keystrokes from a USB keyboard (using an on-board USB to PS/2 converter to convert 
the more complex USB protocol to the simpler PS/2 protocol). 
It converted the data from the keyboard to ASCII characters, and then either converted these ASCII characters 
to their respective pixel representations in 8x12 bitmapped format, using a character lookup table, or performed 
some sort of combinational logic depending on whether the characters were special (such as backspace, new line, shift, and caps lock). 
The color of the pixel representation was determined by the 4 least significant of the 16 on-board switches. 

## Introduction
The digital system constructed in this project was implemented on the Nexys A7-100T FPGA trainer board. 
This board comes equipped with many peripherals and I/O devices, including a USB port, a set of on-board switches, 
a VGA port, 7-segment LED display, etc. Three of these peripherals were used in this digital system: the switches, 
USB port (via the USB to PS/2 converter), and the VGA port. Each of these components comprised a slave module on the AXI4-Lite bus. 
The slave and master modules can be seen in Figure 1. 

**The PS/2 Slave Module: ps2_0**  \
The ps2_0 slave module, which handled input from the keyboard, was responsible for receiving and validating the scancodes generated by the PS/2 keyboard. 
In PS/2 protocol, 11 bits are sent clocked by a keyboard, on the falling edge of the keyboard clock. 
The 11 bits are a Start bit that is always 0, 8 data bits, a parity bit, and a stop bit that is always 1. 
The validity of the data received is verified by the parity bit. Essentially, if there is an even number of 1s in the data bits,
the parity bit should be 1. When a key is pressed, the scancode is sent. When the key is released, the scancode is sent again but preceded by F0. 
If the key is held down, the scan code is repeated. Initially the  PS2_DATA and PS2_CLK ports are set to the Z (high-impedance state). 
Once the keyboard is connected it uses the PS2_DATA port to transmit the scancode one bit at a time, clocked at its own clock rate, using the PS2_CLK port. 
Once a scancode is received and validated, the slave module sends an interrupt signal IRQ_O to the master module’s IRQ_I port. 
An interrupt-driven process is used instead of a polling process because keystrokes are relatively irregular and can be temporally spread apart. 
Therefore it makes more sense for the slave to send an interrupt to the master when it is ready to write, rather than the master regularly poll the slave. 

**The AXI Switches Module: axi_switches_0** \
The axi_switches_0 slave module is relatively simpler than the ps2_0 module. 
It simply sends the switch data as a std_logic_vector of 16 bits to the master via the bus (using the AXI4-Lite protocol). 
It does not need to read anything from an I/O device that uses a separate clock and does not need to validate the data received. \

**The VGA Slave Module: AXI_VGA_Slave_0** \
The AXI_VGA_Slave_0 module is responsible for generating the signals necessary to drive the VGA. 
It takes the write_data from the master module and directly sends it to a block RAM. 
Simultaneously it reads a line from the block ram and, using an internal pixel counter and pixel clock generated from dividing 
the 100MHz system clock (CLK100MHZ). From the block ram it determines what to set VGA_R, VGA_G, and VGA_B, the red, green, and blue pixel signals, respectively.
It also generates VGA_HS and VGA_VS (hsync and vsync) signals to synchronize the VGA and drive the display. \

**The Master Module: text_display_master_0** \
	The text_display_master_0 module is responsible for converting the scancodes to pixel data that it sends to the AXI_VGA_Slave_0 module. 
  The master contains a few major components: MASTER_EXECUTION_PROC, irq_delay, char8x12_lookup_table, scancode2ascii, and pixelcode_to_pixelword. 
	Scancode2ascii and pixelcode_to_pixelword are purely combinational logic circuits. The rest are sequential. Scancode2ascii, as its name implies, 
  converts scancode from the ps2_0 slave and converts it to ASCII. The char8x12_lookup_table module is a sequential circuit that takes the ASCII code and 
  an integer ranging from 0 to 11 (for the 12 lines in an 8x12 character) and outputs an 8-bit pixel code, where a 0 indicates a background pixel and a 1 
  indicates a foreground pixel. The pixelcode_to_pixelword module converts the pixel code output by the lookup table and converts it to a 32-bit word of
  8 pixels (4 bits per pixel). This is the data that will actually be written to block memory and decoded by the AXI_VGA_Slave. These modules deal mainly 
  with the writing protocol. The irq_delay module deals with the read protocol. It takes the interrupt signal sent from the ps2_0 slave module and delays 
  it a bit in order to delay the read from the slave. \
  
The most important component of the entire system is the MASTER_EXECUTION_PROC. It is a finite state machine that is responsible for coordinating reads 
  from the ps2_0 slave and the axi_switches_0 slave and writes to the AXI_VGA_Slave_0 slave. It has the following states: INITMEM, IDLE, SW_READ, INIT_READ, 
  KEYUP_EXTEND_ASSERTION, KEYUP_DATARELEASE, VGA_WRITE, and VGA_ERASE. When the reset button is pressed, the FSM de-asserts all signals and flags and goes to 
  the INITMEM state. \
  
In the INITMEM state, the master continuously issues writes to memory until memory is full. The data written to memory, determined in a different process, 
is lines of background-colored pixels, essentially clearing the screen. Once the RAM is full, the FSM moves to the IDLE state. \
In the IDLE state, if the delayed interrupt signal is asserted, the state switches to the INIT_READ state. Else it moves to the SW_READ state. The SW_READ 
state issues a single read of the axi_switches_0 data and then the state changes back to IDLE. In this way, the states keep switching between IDLE and SW_READ, 
until the interrupt is asserted and it switches to INIT_READ. \

INIT_READ issues a single read of the ps2_0 data, i.e. the scancode, and then the state changes to KEYUP_EXTEND_ASSERTION. Here, depending on the scancode, 
multiple things could happen. If the scancode is an extend code (E0) then the E0_Received flag is asserted and the state goes back to IDLE. 
If it is a break code (F0) then, depending on if the E0_Received flag is asserted it will either assert the F0_Received flag or it will not. 
There is additional functionality on what to do when the scancode for backspace, shift, capslock, or enter is received. \

If backspace is pressed, a BS_pressed flag is asserted as well as the F0_Received flag and the state moves directly to KEYUP_DATARELEASE. 
If enter is pressed, the F0_Received tag is de-asserted so that nothing is written, and the memory address is changed to the beginning of the next line on the screen. 
If capslock is pressed, a CS_pressed flag is asserted and the F0_Received flag is deasserted. A similar thing happens for when shift is pressed, except the flag is 
deasserted if the break code has been received as the Shift_pressed flag should only be asserted when shift is continuously pressed and not upon its release. 
For each of these scancodes the state moves back to IDLE. If none of these keys has been pressed, then, depending on whether the E0_Received flag has been asserted or not, 
the state moves back to IDLE or KEYUP_DATARELEASE. \

In KEYUP_DATARELEASE, if the F0_Received flag has not been asserted, the state moves back to IDLE. Otherwise, it checks a few things. 
If the BS_pressed flag is asserted, the write address is changed to the beginning of the previous character and then the state moves to VGA_ERASE. 
If the Shift_pressed flag has been received, the capital signal is asserted. This signal is used to access capital ascii codes.
If the CS_pressed flag is asserted, the capital signal is toggled. The state then moves to VGA_WRITE. \ 

In the VGA_ERASE state, a single character block is written to memory. The write data in this case is background pixels, essentially clearing the block. 
The address then remains the same and the state moves to IDLE.  \

In the VGA_WRITE state, a single character is written to block memory as well. The write data is the pixel word output from the pixelcode_to_pixelword module. 
The address is then changed to the next character block and then the state goes back to IDLE.