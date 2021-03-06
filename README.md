# pdp11-code

The **simh** application provides simulation of DEC's PDP11 computers.

A PDP11 Programming card may be obtained from: https://archive.org/details/bitsavers_decpdp11PDul75_1582192/page/n1/mode/2up

The latest versions of *simh* may include [Readline](https://en.wikipedia.org/wiki/GNU_Readline) If the version you have does not, then it is advantageous to download and invoke the *rlwrap* utility.

## Testing CSR Status and Looping

One way to move data from a register to, say, the console terminal display is to test the CSR at address 177564 to see if the device is ready to receive a character. Loop until it is, and then move the character to the console buffer at address 177566. For example run the file:

* console-tstb-character

The code in this file is:
```
echo TSTB Address of Console Display CSR 177564
d 1000 105737
d 1002 177564
echo Branch if Plus. i.e. If no 000200 to indicated "ready".
d 1004 100375

echo MOV #101 octal i.e. Upper case ASCII A to Console buffer 177566
d 1006 112737
d 1010 000101
d 1012 177566

echo Do the same as above but move the number 102 octal. i.e. The letter B.
d 1014 105737
d 1016 177564
d 1020 100375
d 1022 112737
d 1024 000102
d 1026 177566

echo HALT
d 1030 000000

g 1000
```
The console displays:
```
$ rlwrap pdp11 console-tstb-character 
PDP-11 simulator V3.8-1
TSTB Address of Console Display CSR 177564
Branch if Plus. i.e. If no 000200 to indicated "ready".
MOV #101 octal i.e. Upper case ASCII A to Console buffer 177566
Do the same as above but move the number 102 octal. i.e. letter B.
A
HALT instruction, PC: 001032 (HALT)
sim> 
```
Note that the "B" is not displayed as the simulator performed the HALT in the code before it had delivered the "B" character out to the console. This timing issue may be overcome by adding a delay before halting. E.g.
```
echo Add delay before halting
d 1030 105737
d 1032 177564
d 1034 100375

echo HALT
d 1036 000000
```

The console now displays...
```
Do the same as above but move the number 102 octal. i.e. letter B.
Add delay before halting
HALT
AB
HALT instruction, PC: 001040 (HALT)
```

## Interrupt Driven Routines.

PDP11 code may be enhanced from using looping routines to test if devices are ready to send/receive data, to using *interrupt driven routines*. 

## Simple Console Keyboard routine.

This section explains the following files:
* interrupt-overview-keyboard-1
* interrupt-overview-keyboard-2

To demonstrate *interrupt driven routines* a program will wait for a key to be typed on the console keyboard. When it's typed the keyboard interrupt is triggered and then code is executed to immediately handle retrieving a character from the keyboard buffer.  

For these programs the starting address of 1000 has been selected. Thus program script files contain g for go and 1000 to start at address 1000:
```
g 1000
```

An initial piece of housekeeping on starting a program is to set the Stack Pointer (SP). The SP is General Purpose Register 6 and address 1000 is the address it's initially been selected to contain. 
```
MOV #1000, R6
012706
001000
```
The stack never uses address 1000. It works downwards, so the first entry on the stack will be 776, then 774, then 772, etc.

The program needs to set the address and PSW to use for a routine to be executed when an interrupt occurs. For example addresses 60 and 62 are used for the keyboard of the console terminal. Into 60 is placed the address of 2000, where the keyboard input routine starts. Into 62 is placed the PSW that the keyboard routine will start execution at. In the case of the console keyboard, 200 is used so that it will set a priority of 4. Settings priority levels for the PSW is as follows: 7 = 340, 6 = 300, 5 = 240, 4 = 200, 3 = 140, 2 = 100, 1 = 40
```
MOV #2000 to Address 60
012737
002000
000060

MOV #200 (priority 4) be the future PSW at Address 62
012737
000200
000062
```
The next part of the program that started at address 1000 is to set the priority in the PSW to be less that 4 so an interrupt from the keyboard is at a higher level and may occur. By default simh/pdp11 seems to start with the PSW at 340 which is priority 7.
```
MOV # 140 (priority 3) to PSW address of 177776
012737
000140
177776
```
Now the keyboards interrupt enabled bit 6 needs to be set.
```
Mov #100 to Keyboard CSR 177560
012737
000100
177560
```
After this there needs to be a WAIT for interrupt, where the program waits until someone types a key on the keyboard.
```
WAIT for interrupt
000001
000000
```
For the moment make the next instruction after the WAIT to be a HALT 000000.

The keyboards interrupt routine will be written soon, in the meantime a HALT instruction will be placed at the start of the routine at address 2000.
```
d 2000 000000
```

With address 2000 just contains a HALT, 000000, at the moment, then here is what happens when the code is run from address 1000:
* Housekeeping is performed and everything is set as above
* The program gets to the WAIT instruction and waits for a key on the console to be pressed. 
* When the key is pressed the keyboard hardware interrupt occurs. 
* The current PSW of 140 is placed on the stack at address 776.
* The updated PC (after the WAIT instruction) is placed on the stack at address 774. 
* The value of 2000 at interrupt vector address of 60 is loaded into the Program Counter (PC) 
* The value of 200 (priority 4) at interrupt vector address of 62 is loaded in the PSW.

To run this program enter...
```
$ rlwrap pdp11 interrupt-overview-keyboard-1
```
... the program starts at 1000 and then WAIT's for the Interrupt. Type a key on the keyboard. The interrupt is processed and then execution of code continues at address 2000 where a HALT instruction is found. It halts and displays the updated PC of 2002. You can then examine the status of registers, etc. with commands like:
```
e PC
e PSW
e SP
e 770:1000
```
...which will provide output like this...
```
sim> e PC
PC:	002002
sim> e PSW
PSW:	000200
sim> e SP
SP:	000774
sim> e 770:1000
770:	000000
772:	000000
774:	001036
776:	000140
1000:	012706
```

Now write some code at address 2000 to capture some info and return from the interrupt. E.g.
```
MOV the contents of Keyboard buffer to Register 0
013700
177562

MOV stack address 776 and 774 to R1 and R2 for having a look later
013701
000776

013702
000774

Lets see what the SP and PSW are set to: MOV R6 to R3 and MOV 177776 to R4
010603
013704
177776

Finished. So, Return from interrupt (RTI)

RTI
000002
```

When the RTI occurs the Stack will be read and the address of the WAIT for interrupt plus 2, will be retrieved and fed to the Program counter, along with the stack providing the PSW to change back to priority 3.

The stack will continue to contain the address of the RTI + 2 and the PSW with a priority of 4, however the Stack Pointer will move back to 1000.

To run this program enter...
```
$ rlwrap pdp11 interrupt-overview-keyboard-2
```

The code total code of the program is as follows:
```
echo MOV #1000 to the Stack Pointer.
d 1000 012706
d 1002 001000 

echo MOV #2000 to Address 60
d 1004 012737
d 1006 002000
d 1010 000060

echo MOV #200 (priority 4) be the future PSW at Address 62
d 1012 012737
d 1014 000200
d 1016 000062

MOV # 140 (priority 3) to PSW address of 177776
d 1020 012737
d 1022 000140
d 1024 177776

echo MOV #100 to Keyboard CSR 177560
d 1026 012737
d 1030 000100
d 1032 177560

echo WAIT for interrupt.
d 1034 000001

echo HALT (for now) when returning from an interrupt routine.
d 1036 000000

echo Keyboard interrupt routine. Once a key has been typed.
echo MOV contents of address 177562 to R0
d 2000 013700
d 2002 177562

echo Lets also get the current Stack contents into R1 and R2
d 2004 013701
d 2006 000776
d 2010 013702
d 2012 000774

echo Lets see what the SP and PSW are set to: MOV R6 to R3 and MOV 177776 to R4
d 2014 010603
d 2016 013704
d 2020 177776

echo RTI this is to the HALT at address 1036
d 2022 000002

echo Start at address 1000
g 1000
```

The program is now run, starting at address 1000. It pauses waiting for a key to be typed. Once a key has been typed, for example a Capital A, which is 101 in octal, then the program halts with:
```
HALT instruction, PC: 001040 (HALT)
```
Now take a look around...
```
sim> e R0
R0:	000101 <-- This is the Capital "A" in R0 taken from 177562 keyboard buffer.
sim> e r1
R1:	000140 <-- The Stack pointer for the PSW when returning from Keyboard routine.
sim> e r2
R2:	001036 <-- The Stack pointer for the PC when returning from Keyboard routine.
sim> e r3
R3:	000774 <-- Address in the Stack Pointer (SP) to use when next removing from stack.
sim> e r4
R4:	000200 <-- PSW is at priority 4 when running the keyboard routine.
sim> e 770:1000 <-- After return from keyboard routine stack remains unchanged
770:	000000
772:	000000
774:	001036
776:	000140
1000:	012706
sim> e SP 
SP:	001000 <-- However the Stack Pointer (SP) has now changed from 774 to 1000
sim> e PSW
PSW:	000140 <-- Has dropped from priority 4 = 200, to priority 3 = 140
sim> e PC
PC:	001040 <-- Halted at the address after the WAIT for interrupt.
sim> 
```
To perform the above run:
```
$ rlwrap pdp11 interrupt-overview-keyboard
```

## Simple Console Keyboard routine with display of characters on the console

The following program is used:
* interrupt-overview-keyboard-with-echo

To echo the keyboard keys typed, then, at this stage, it will be done with a TSTB loop rather than an interrupt routine.

The code above is enhanced as follows:
```
echo WAIT for interrupt.
d 1034 000001

echo HALT (for now) when returning from an interrupt routine.
echo d 1036 000000

echo Change HALT for a Branch back one instruction to the WAIT. To continue looping.
d 1036 000776

echo Test for Control q exit. CMPB 21 (Ctrl q) with R0. Branch to HALT
d 2022 122700
d 2024 000021
d 2026 001406

echo output R0 to the Console terminal display TSTB CSR and BPL. MOVB R0 to Buffer.
d 2030 105737
d 2032 177564
d 2034 100375
d 2036 110037
d 2040 177566

echo RTI to return to address 1036, which branchs back one instruction for the WAIT.
d 2042 000002

echo HALT for Ctrl q
d 2044 000000
```
To perform the above:
```
$ rlwrap pdp11 interrupt-overview-keyboard-with-echo
```
...to exit from this program type Control q.
 
## Output a Message to the Console Display

On the PDP11 the addresses for output to the console terminal display are 177546 for the CSR and 177566 for the Buffer. The interrupt vectors for the console output are 64 for the address of the output routine and 66 for the PSW to be used when commencing execution of the output routine.

In the case of the keyboard interrupt, you control when this commences, as you are the person that types the first key on the keyboard that generates the first interrupt.

In generating an interrupt for output to the console terminal display this is determined by the device. If the device is ready to output a character (i.e. a 200 is the value in its CSR of 177564) and you set Bit 6, the interrupt bit, by moving a 100 to the CSR. Then an interrupt will immediately occur before the next instruction can be executed and before your program has reached the WAIT for Interrupt instruction.

This may be undesirable, as the program may not have reached a stage of having its data set up to send to the console display.

One method to overcome this is to send a NULL character (000) to the console display when this first interrupt occurs. While this is being done, the program can continue executing code and set up transfer of the message via the WAIT for interrupt process.

Please review and run the attached program:
* interrupt-driven-console-output
 
Objectives of this program are:

* The message is stored from location 4000 onwards. Each 16 bit word of the message contains two bytes and each byte is an ASCII character. 
* The main program starts at address location 1000. 
* After housekeeping procedures the number 4000, the start address of the message is moved to R0.
* The contents of the R0 register is then used to get a byte of data from the address it points to, which it stores in R1. R0 then auto-increments by 1, so its pointing to the next byte of data.
* Once an updated byte of data is placed in R1, the interrupt bit for the Console output CSR is turned on.
* The program then executes a WAIT instruction and pauses until the console device comes "ready" and generates an interrupt.
* The interrupt occurs and the routine commences execution at loacation 3000.
* The first task is to turn off further interupts by the console, which is done by sending a 1 to the CSR. If this is not done, then the same character may repeatedly be output to the console.
* The character in R1 is then moverd to the Console output buffer 177566 and is displayed.
* A return from interrupt then occurs. 
* The next character in the message is retrieved and after turning interrupts back on a WAIT is performed until the device it "ready" for the next character and performs an interupt.
* Each time a check is performed on R1 to see if it contains zero. When it does the program HALTs as the message has been completely sent to the console display.

 ## Stop Watch
 
A device interrupt driven Stop Watch. After initial setup the program encounters the one WAIT instruction in the program. Device interrupts may be performed by the KW11 clock, the console keyboard and the console display. After each device has completed its interrupt routine a RTI (Return from Interrupt) is performed to go back to the WAIT instruction. Please run the program:

* stop-watch

The console display will be like this...

```
$ rlwrap pdp11 stop-watch

PDP-11 simulator V3.8-1
Disabling CR
Disabling XQ

PDP11 Stop Watch.
Enter key to toggle Start and Stop.
Spacebar to Reset timer to zero.
Escape key to HALT.

 00:07:15.9
```


 
 
 
