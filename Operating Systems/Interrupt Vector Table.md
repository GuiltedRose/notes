Interrupts are like sub-routines, but you don't need to know the memory address to invoke them. When you invoke an interrupt the processor gets interrupted, the old state is then saved on the stack, and then the interrupt is executed. The old state saves things like a return address.

The interrupt vector table explains where these interrupts are in memory. We have 256 interrupt handlers, and each entry in our table is 4 bytes (Offset:Segment). They are also in numerical order within the table.

| Offset  | Segment | Offset  | Segment | Offset  | Segment | Offset  | Segment |
| :-----: | :-----: | ------- | ------- | ------- | ------- | ------- | ------- |
|  0x00   |  0x7c0  | 0x8d00  | 0x00    | 0x00    | 0x8d0   | 0x7c0   | 0x587   |
| 2 bytes | 2 bytes | 2 bytes | 2 bytes | 2 bytes | 2 bytes | 2 bytes | 2 bytes |
	Interrupt 0             Interrupt 1                     interrupt 2                    interrupt 3
	 0x00                         0x04                                 0x08                         0x16

What is the offset of 0x13?
0x13 * 0x04 = 0x46 or 76h (the h dictates decimal notation in computing). 
"Int 0x13" processor looks at offset 76 in RAM
76h - 77h = Offset (0000)
78h - 79h = Segment (01D8h)

(You can find the interrupt list in the tools section of [[Building an Operating System]])

If we look at Interrupt 0 in our table, we can see it looks familiar. Basically what happens is we tell the processor "Set our offset to 0x00". Then we tell it to "Jump to 0x7c0 to start an interrupt". The interrupt then executes until it's terminated by an iret (interrupt return) instruction. This is how they kinda act as a special sub-routine that are called by numbers instead of memory addresses.

## Handling Interrupts:

```asm
ORG 0
BITS 16
_start:
	jmp short start
	nop

times 33 db 0

start:
	jmp 0x7c0:step2

handle_zero:
	mov ah, 0eh
	mov al, 'A'
	mov bx, 0x00
	int 0x10
	iret


step2:
	cli ; Clear Interrupts
	mov ax, 0x7c0
	mov ds, ax
	mov es, ax
	mov ax, 0x00
	mov ss, ax
	mov sp, 0x7c00
	sti ; Enables Interrupts
	mov word[ss:0x00], handle_zero ; moves word to stack
	mov word[ss:0x02], 0x7c0
	
	int 0
	
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
In order to change our data from running our bootloader without destroying the underlying code & manually rewriting it, we can move our "Hello World!" data to the stack to then be moved to a new memory address in order to reuse the 1st byte of RAM again.

If we recompile & run the program it should now be changed.

`nasm -f bin ./boot.asm -o ./boot.bin`
`qemu-system-x86_64 -hda ./boot.bin`
![Interrupt 0](https://github.com/GuiltedRose/notes/blob/main/pictures/interrupt0.png?raw=true)
We can see that our code updated correctly as `AHello World!`. If you see this it means you correctly changed the memory address of interrupt 0. Instead of calling `int 0`, we can also do this:
```
mov ax, 0x00
div ax
```
This will still cause our bootloader to execute interrupt 0 because it's a divide by zero exception.

```asm
ORG 0
BITS 16
_start:
	jmp short start
	nop

times 33 db 0

start:
	jmp 0x7c0:step2

handle_zero:
	mov ah, 0eh
	mov al, 'A'
	mov bx, 0x00
	int 0x10
	iret

handle_one:
	mov ah, 0eh
	mov al, 'V'
	mov bx, 0x00
	int 0x10
	iret


step2:
	cli ; Clear Interrupts
	mov ax, 0x7c0
	mov ds, ax
	mov es, ax
	mov ax, 0x00
	mov ss, ax
	mov sp, 0x7c00
	sti ; Enables Interrupts
	mov word[ss:0x00], handle_zero ; moves word to stack
	mov word[ss:0x02], 0x7c0
	
	mov word[ss:0x04], handle_one
	mov word[ss:0x06], 0x7c0
	
	int 1
	
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
Something you may notice, is that every single time we add an interrupt we change our origin 2 bytes after the executed interrupt. This is because it takes up 4 bytes, so we then need to move the stacked data 4 bytes after it. (You don't need to compile this, I just wrote it to show the trend in offsets.)
If you do run it for your own learning you should see this output.
![Interrupt 1](https://github.com/GuiltedRose/notes/blob/main/pictures/interrupt1.png?raw=true))
[[Disk Access]]
