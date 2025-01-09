* boot.asm file is created for the bootloader.
```asm
ORG 0x7c00
BITS 16

start:
	mov ah, 0eh
	mov al, 'A'
	mov bx, 0
	int 0x10
	
	jmp $
	
times 510-($ - $$) db 0
dw 0xAA55
```
This should print the character "A" to our screen when ran via QEMU.
When developing for an operating system we should technically originate from 0 then jump to 0x7c00, but for the sake of this small bit of code it's ok to originate from 0x7c00.
We then set our code to run in 16 bit mode for the boot process.
after that we execute a bit of code that will loop within itself to print "A" after boot.

`times 510-($ - $$) db 0` will fill the rest of the data we allocated with 0's until it's full.
`dw 0xAA55` will then tell our program to enter boot mode. The reason it's backwards is because x86 processors are "little endian". This means the bytes are flipped for words, and the dw portion of our code tells it that it's 2 sectors long.
We can now assemble the code with this line:
`nasm -f bin ./boot.asm -o ./boot.bin`
You can run it with this command: `qemu-system-x86_64 -hda ./boot.bin`.
[[https://github.com/GuiltedRose/notes/blob/main/pictures/kernel-print-a.png]]
The image is the result we should get from QEMU after running the binary file. (taken from my system).

```asm
ORG 0x7c00
BITS 16

start:
	mov si, message
	call print	
	
	jmp $

print:
	mov bx, 0
.loop:
	lodsb
	cmp al, 0
	je .done
	call print_char
	jmp .loop
.done:
	ret

print_char:
	mov ah, 0eh
	int 0x10
	ret
message: db 'Hello World!', 0

times 510-($ - $$) db 0
dw 0xAA55
```
We modified our code to look something like this. We did this so we can display more than one character at a time. We call the si register this time, because it points to the beginning of a memory address. In our case it points to 'Hello World!' in the db register. We then call our print label that holds most of our logic. Since we are using NASM for Intel x86_64 processors we can call sub-labels within other labels. This is where `.loop` & `.done` take place in our code. 

At the beginning of our loop sub-label we use `lodsb` to load data in from the si register. We then compare if the al register is equal to zero, if not we jump back to the top of our loop until we return from the sub routine.

Looking at the `print_char` sub routine we can see it's the same code we used before, we just moved it out of the way in order to use it to step through the preexisting data in memory('Hello World!').

We can now recompile our code with NASM & get our new binary file to run through QEMU.
![[https://github.com/GuiltedRose/notes/blob/main/pictures/kernel-hello-world.png]]
## Understanding Real Mode:
Real mode is the starting point for **ALL** x86_64 processors. It only has 1 megabyte of RAM accessible at this time, until you boot into Protected Mode.
Real mode has no security.
It can only run 16 bits at a single time.
Memory access is done via the segmentation memory model.
When the processor runs in Real Mode it acts similar to a processor from the 1970's such as the 8086 processor from Intel (documentation for this processor will be linked for the purposes of real mode.) 

**ALL** code in real mode is **REQUIRED** to be 16 bits. This can be changed via overrides but we are avoiding it for our use case.

No security means our memory & hardware are both at risk to the end user to cause a ton of damage and kill the kernel's process thus stopping the machine or burning out our hardware.

Think of our kernel as having 2 layers:
* one layer is for hardware to interface with the kernel this is typically called kernel-land.
* the other is known as user-land which allows the user to interact with the kernel and benefit from the hardware without causing a crash or burning out our components.

Because real mode requires us to be in 16-bit mode we only have access to 8-bit & 16-bit registers.
This means we can only request memory address offsets of up to 65535 for our given segment.
This means we can't go higher than 65KB.

## Segmentation Memory Model:
* Memory is accessed by a segment & an offset.
* Programs can be loaded in different areas of memory, but run without any problems.
* Multiple segments are used via the use of segment registers.
### 8086 Segment Registers:
* CS -> code segment
* SS -> stack segment
* DS -> data segment
* ES -> extra segment

We use segment registers in order to access memory in RAM.
We can take a segment and multiply it by 16 and add an offset in order to calculate an absolute position for an address in RAM.

For example:
*  cs = 0x7c0
* org = 0
* we would take (0x7c0 * 16) and get 0x7c00
* Then we  add the offset (0x7c00 + 0) to get 0x7c00. This is because our offset is dictated by the assembly origin of a file in this case it's 0.

Different instruction use different registers.
"lodsb" uses the DS:SI register combination.
An example is this:
```asm
ORG 0
mov ax, 0x7c0
mov ds, ax
mov si, 0x1F
loadsb
```
This code won't do anything, but we can use it to visualize what's going on with the changes.
0x7c0 * 16 = 0x7c00
0x7c00 + 0x1f = 0x7c1f

This tells our program to grab the byte in the segment 0x7c1f.

The reason we can load multiple programs at the same time in different memory addresses is due to them being saved in different registers. For example 0x7c00 can switch to 0x7d00 to execute something completely different and move back like nothing happened.

Multiple segments are available through the use of segment registers.
* `mov byte al, [es:32]` -> Extra Segment
* `mov byte al, [ds:826]` -> Data Segment
* `move byte al, [ss:231]` -> Stack Segment
* these are all taking the same data to a different register.
### Stack Segment:
* SS = 0x00 -> Stack Segment
* SP = 0x7c00 -> Stack Pointer
* push 0xffff
* decrements stack pointer by 2
* stack pointer = 0x7FE
* set 0x7BFE-0x7BFF to 0xffff
The stack pointer points to a place in memory, and when doing stack operations it deals with the memory address we provide. The absolute address is given by taking SS(Stack Segment) * 16 + SP(Stack Pointer). 
==**Note: Only 2 bytes are pushed here because we are in a 16-bit mode. If we ran in 32-bits then 4 would be pushed, and if it's 64-bit 8 bytes would be pushed.**==

## Back to Our Bootloader:
```asm
ORG 0
BITS 16

start:
	cli ; Clear Interrupts
	mov ax, 0x7c0
	mov ds, ax
	mov es, ax
	sti ; Enables Interrupts
	mov si, message
	call print	
	
	jmp $

print:
	mov bx, 0
.loop:
	lodsb
	cmp al, 0
	je .done
	call print_char
	jmp .loop
.done:
	ret

print_char:
	mov ah, 0eh
	int 0x10
	ret
message: db 'Hello World!', 0

times 510-($ - $$) db 0
dw 0xAA55
```
We set our origin to 0 to ensure that most operating systems are able to boot from our assembly code. We can't be certain that the BIOS of all systems sets the origin to 0x7c00 by default.
For example if we set ours to 0x7c00, and the BIOS uses 0, we will do the calculations for booting incorrectly.
```
ORG 0x7c00
BITS 16
0x7c0 + 16 = 0x7c00
0x7c00 + 0x7c00 ; This is the incorrect math...
```
So instead of ending with 0x7c00 we'd end with a completely different segment. We'll get 0xF800 instead.
