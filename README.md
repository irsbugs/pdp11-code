# pdp11-code

The **simh** application provides simulation of DEC's PDP11 computers.

A PDP11 Programming card may be obtained from: https://archive.org/details/bitsavers_decpdp11PDul75_1582192/page/n1/mode/2up


## Testing CSR Status and Looping

One way to move data from a register to, say, the console terminal display is to test the CSR at address 177564 to see if its ready to receive a character. Loop until it is, and when it is, then move the character to the Console buffer at address 177566. For example run the file:

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

echo Do the same as above but move the number 102 octal. i.e. letter B.
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

## Interrupt Driven Routines.

PDP11 code may be modified from using looping routines to test if devices are ready to send/receive data, to using *interrupt driven routines*.

For these programs the starting address of 1000 has been selected. An initial piece of housekeeping on starting a program is to set the Stack Pointer (SP). The SP is General Purpose Register 6 and address 1000 is the address its initially been selected to contain. 
```
MOV #1000, R6
012706
001000
```

The stack never uses address 1000. It works downwards, so the first entry on the stack will be 776, then 774, then 772, etc.

I then get my program to set the address and PSW to use for a routine to be executed when an interrupt occurs. For example addresses 60 and 62 are used for the keyboard of the console terminal. Into 60 I put the address of 2000, when my keyboard input routine starts. Into 62 I put the PSW I want the keyboard routine to use. I enter 200 so that it has a priority of 4.

MOV #2000 to Address 60
012737
002000
000060

MOV #200 (priority 4) be the future PSW at Address 62
012737
000200
000062

Then next part of my program that started at address 1000 is to set the priority of my program in the PSW to be less that 4 so an interrupt from the keyboard is at a higher level and may occur. By default simh/pdp11 seems to start with the PSW at 340 which is priority 7.

MOV # 140 (priority 3) to PSW address of 177776
012737
000140
177776

Now the keyboards interrupt enabled bit 6 needs to be set.
Mov #100 to Keyboard CSR 177560
012737
000100
177560

After this there needs to be a Wait for interrupt, where the program waits until someone types a key on the keyboard.

WAIT for interrupt
000001

For the moment make the next instruction after the WAIT to be a HALT 000000.

d 2000 000000


Assuming address 2000 just contains a halt 000000 at the moment, then heres what happens when the code is run from address 1000.

Everything is set as above and the program gets to the WAIT instruction and waits for a key to be pressed. When the key is pressed the keyboard interrupt occurs. The the Current PSW of 140 is placed on the stack at address 776 and the updated PC (after the WAIT instruction) is placed on the stack at address 774. 

The value of 2000 at interrupt vector address of 60 is loaded into the Program Counter (PC) and the vaMov #100 to Keyboard CSR 177560
012737
000100
177560lue of 200 (priority 4) at interrupt vector address of 62 is loaded in the PSW.

The program then starts executing code at address 2000 and finds a HALT instruction. It halts and displays the updated PC of 2002. You can then examine things with commans like:

e PC
e PSW
e SP
e 770:1000

Now write some code at address 2000 to capture some info and return from the interrupt. E.g.

MOV the contents of Keyboard buffer to Register 0
013700
177562

MOV stack address 776 and 774 to R1 and R2 for having a look later
013701
000776

013702
000774

Now Return from interrupt (RTI)

RTI
000002

When the RTI occurs the Stack will be read and the address of the WAIT for interrupt plus 2, will be retrieved and fed to the Program counter, along with the stack providiing the PSW to change back to priority 3.

The stack will now contain the address of the RTI + 2 and the PSW with a priority of 4.

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

Start at address 1000
Keyboard interrupt routine. Once a key has been typed

... A Capital A was typed...

HALT instruction, PC: 001040 (HALT)

Now take a look around...
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
sim> e SP <-- However the Stack Pointer (SP) is now changed from 774 to 1000
SP:	001000
sim> e PSW <-- Have dropped from priority 4 to priority 3
PSW:	000140
sim> e PC
PC:	001040 <-- Halted at the after the WAIT for interrupt.
sim> 

To perfrom the above run:

$ rlwrap pdp11 interrupt-overview-keyboard

To echo the keyboard keys typed, then (at this stage) it is done with a TSTB loop rather than an interrupt routine.

The code above is enhanced as follows:

echo WAIT for interrupt.
d 1034 000001

echo HALT (for now) when returning from an interrupt routine.
echo d 1036 000000

echo Change HALT for a Branch back one instruction to the WAIT. To continue looping.
d 1036 000776


echo Test for Control Q exit. CMPB 21 (Ctrl Q) with R0. Branch to HALT
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

echo HALT for Ctrl Q
d 2044 

To perform the above:

$ rlwrap pdp11 interrupt-overview-keyboard-with-echo


 
 
