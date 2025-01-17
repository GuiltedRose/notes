## Selector Memory Scheme:
* Segment registers become selector registers
* Selectors point to data structures that describe memory ranges and the permissions (ring level) require to access a given range.
## Paging Memory Scheme:
* Memory is virtual & what you address can point to somewhere entirely different in memory. -> Allows multiple apps to be running without being aware of each other while using the same memory addresses.
* Memory protection is easier to control.
* Paging is the most popular choice for memory schemes with kernels/operating systems.
* ALL page memory has to be dividable by 4096 no remainders or it won't work.
## 4GB Addressable Memory:
This gives us access to 32-bit instructions, and can easily work with 32-bit registers. We can address up to 4GB of memory at any time and are no longer limited to the 1MB of memory provided by real mode.

## Development:
```asm
ORG 0x7c00
BITS 16

CODE_SEG equ gdt_code - gdt_start
DATA_SEG equ gdt_data - gdt_start

_start:
	jmp short start
	nop

times 33 db 0

start:
	jmp 0:step2

step2:
	cli ; Clear Interrupts
	mov ax, 0x00
	mov ds, ax
	mov es, ax
	mov ss, ax
	mov sp, 0x7c00
	sti ; Enables Interrupts
		
.load_protected:
	cli
	lgdt[gdt_discriptor]
	mov eax, cr0
	or eax, 0x1
	mov cr0, eax
	jmp CODE_SEG:load32

; GDT
gdt_start:
gdt_null:
	dd 0x0
	dd 0x0

; offset 0x8
gdt_code:     ; CS SHOULD POINT TO THIS
	dw 0xffff ; Segment limit first 0-15 bits
	dw 0      ; Base first 0-15 bits
	db 0      ; Base 16-23 bits
	db 0x9a   ; Access byte
	db 11001111b ; High 4 bit and low 4 bit flags
	db 0      ; Base 24-31 bits

; offset 0x10
gdt_data:     ; DS, SS, ES, FS, GS
	dw 0xffff ; Segment limit first 0-15 bits
	dw 0      ; Base first 0-15 bits
	db 0      ; Base 16-23 bits
	db 0x92   ; Access byte
	db 11001111b ; High 4 bit and low 4 bit flags
	db 0      ; Base 24-31 bits

gdt_end:

gdt_discriptor:
	dw gdt_end - gdt_start-1
	dd gdt_start
	
[BITS 32]
load32:
	mov ax, DATA_SEG
	mov ds, ax
	mov es, ax
	mov fs, ax
	mov gs, ax
	mov ss, ax
	mov ebp, 0x00200000
	mov esp, ebp
	jmp $

times 510-($ - $$) db 0
dw 0xAA55
```

In order to set up protected mode, we need to clear out all of our testing code. We also had to set up a gdt table in order to reach 32 bit mode. This will give us full access to the 32-bit registers we need for our kernel testing purposes. This is a more modern booting system, you should also check out mbr for the DOS way of doing things (older computers need this). We will support both at some point. (I also want to support 64-bit computing). **Note: I will not be testing code from this point forward in my notes, ==it's still getting tested.**==

## Adding the A20 Line:

```asm
ORG 0x7c00
BITS 16

CODE_SEG equ gdt_code - gdt_start
DATA_SEG equ gdt_data - gdt_start

_start:
	jmp short start
	nop

times 33 db 0

start:
	jmp 0:step2

step2:
	cli ; Clear Interrupts
	mov ax, 0x00
	mov ds, ax
	mov es, ax
	mov ss, ax
	mov sp, 0x7c00
	sti ; Enables Interrupts
		
.load_protected:
	cli
	lgdt[gdt_discriptor]
	mov eax, cr0
	or eax, 0x1
	mov cr0, eax
	jmp CODE_SEG:load32

; GDT
gdt_start:
gdt_null:
	dd 0x0
	dd 0x0

; offset 0x8
gdt_code:     ; CS SHOULD POINT TO THIS
	dw 0xffff ; Segment limit first 0-15 bits
	dw 0      ; Base first 0-15 bits
	db 0      ; Base 16-23 bits
	db 0x9a   ; Access byte
	db 11001111b ; High 4 bit and low 4 bit flags
	db 0      ; Base 24-31 bits

; offset 0x10
gdt_data:     ; DS, SS, ES, FS, GS
	dw 0xffff ; Segment limit first 0-15 bits
	dw 0      ; Base first 0-15 bits
	db 0      ; Base 16-23 bits
	db 0x92   ; Access byte
	db 11001111b ; High 4 bit and low 4 bit flags
	db 0      ; Base 24-31 bits

gdt_end:

gdt_discriptor:
	dw gdt_end - gdt_start-1
	dd gdt_start
	
[BITS 32]
load32:
	mov ax, DATA_SEG
	mov ds, ax
	mov es, ax
	mov fs, ax
	mov gs, ax
	mov ss, ax
	mov ebp, 0x00200000
	mov esp, ebp
	
	; Enabling A20 Line
	in al, 0x92
	or al, 2
	out 0x92, al
	jmp $

times 510-($ - $$) db 0
dw 0xAA55
```
This is needed in order to access the 21st byte in any memory source, without it this byte is completely inaccessible and means we have 0 keyboard support from what I have read on it. Obviously for an operating system this is just as bad as not being able to boot. We can move onto [[Cross Compilers]] for now.

## Refactoring the Code:
We created a few new files called:
`build.sh`, `kernel.asm`, and `linker.ld`
We started our entire refactor by extracting our original protected mode code into `kernel.asm`.

```asm
; Kernel.asm
[BITS 32]
global _start
CODE_SEG equ 0x00
DATA_SEG equ 0x10

_start:
	mov ax, DATA_SEG
	mov ds, ax
	mov es, ax
	mov fs, ax
	mov gs, ax
	mov ss, ax
	mov ebp, 0x00200000
	mov esp, ebp

; Enabling A20 Line
	in al, 0x92
	or al, 2
	out 0x92, al
	jmp $
```
We set it up this way so that our linker will link the file at the end of our bootloader at the 1MB point.
```ld
ENTRY(_start)
OUTPUT_FORMAT(binary)
SECTIONS
{

	. = 1M;
	.text :
	{
		*(.text)
	}
	
	.rodata :
	{
		*(.rodata)
	}
	
	.data :
	{
		*(.data)
	}
	
	.bss :
	{
		*(COMMON)
		*(.bss)
	}
}
```
This is our linker; it's only purpose is to create object files with our make script.
```asm
; boot.asm
ORG 0x7c00
BITS 16

CODE_SEG equ gdt_code - gdt_start
DATA_SEG equ gdt_data - gdt_start

_start:
	jmp short start
	nop  
	
times 33 db 0

start:
	jmp 0:step2

step2:
	cli ; Clear Interrupts
	mov ax, 0x00
	mov ds, ax
	mov es, ax
	mov ss, ax
	mov sp, 0x7c00
	sti ; Enables Interrupts
	
.load_protected:
	cli
	lgdt[gdt_discriptor]
	mov eax, cr0
	or eax, 0x1
	mov cr0, eax
	jmp CODE_SEG:load32
	
; GDT
gdt_start:
gdt_null:
	dd 0x0
	dd 0x0
	
; offset 0x8
gdt_code: ; CS SHOULD POINT TO THIS
	dw 0xffff ; Segment limit first 0-15 bits
	dw 0 ; Base first 0-15 bits
	db 0 ; Base 16-23 bits
	db 0x9a ; Access byte
	db 11001111b ; High 4 bit and low 4 bit flags
	db 0 ; Base 24-31 bits

; offset 0x10
gdt_data: ; DS, SS, ES, FS, GS
	dw 0xffff ; Segment limit first 0-15 bits
	dw 0 ; Base first 0-15 bits
	db 0 ; Base 16-23 bits
	db 0x92 ; Access byte
	db 11001111b ; High 4 bit and low 4 bit flags
	db 0 ; Base 24-31 bits
	
gdt_end:

gdt_discriptor:
	dw gdt_end - gdt_start-1
	dd gdt_start

[BITS 32]
load32:
	mov eax, 1
	mov ecx, 100
	mov edi, 0x100000
	call ata_lba_read
	jmp CODE_SEG:0x100000

ata_lba_read:
	mov ebx, eax, ; Backup the LBA
	; Send the highest 8 bits of the LBA to hard disk controller
	shr eax, 24
	or eax, 0xE0 ; Selects master's drive
	mov dx, 0x1F6
	out dx, al
	; Finished sending the highest 8 bits of the LBA

	; Send total sectors to read
	mov eax, ecx
	mov dx, 0x1F2
	out dx, al
	; Finished sending total sectors to read

	; Send more bits to LBA
	mov eax, ebx ; Restore backup LBA
	mov dx, 0x1F3
	out dx, al
	; Finished sending more bits to LBA

	; Send more bits to LBA
	mov dx, 0x1F4
	mov eax, ebx ; Restore backup LBA
	shr eax, 8
	out dx, al
	; Finished sending more bits to LBA

	; Send upper 16 bits of LBA
	mov dx, 0x1F5
	mov eax, ebx ; Restore backup LBA
	shr eax, 16
	out dx, al
	; Finished sending upper 16 bits of LBA

	mov dx, 0x1f7
	mov al, 0x20
	out dx, al
	
	; Read all sectors into memory
.next_sector:
	push ecx
	
; Checking if we need to read
.try_again:
	mov dx, 0x1f7
	in al, dx
	test al, 8
	jz .try_again
; Need to read 256 words at a time
	mov ecx, 256
	mov dx, 0x1F0
	rep insw
	pop ecx
	loop .next_sector
	; End of reading sectors into memory
	ret

times 510-($ - $$) db 0

dw 0xAA55
```
This is the modified `boot.asm` script that now contains a lot of very important ATA code. This reads our kernel size into memory so we can ensure it's being loaded at the correct memory address. ==**NOTE: we do not need to reset the LBA each time (eax), we just do it to ensure it's not getting corrupt during the boot process.**==
The ATA code is written like this because we are using the LBA format (logical block addressing); this requires the data to be read in a linear fashion. The out statement is taking our inputs out to the motherboard's ATA controller, this is essentially an LBA driver.

The last part of our refactor is our Makefile. 
```Makefile
FILES = ./build/kernel.asm.o

all: ./bin/boot.bin ./bin/kernel.bin
	rm -rf ./bin/os.bin
	dd if=./bin/boot.bin >> ./bin/os.bin
	dd if=./bin/kernel.bin >> ./bin/os.bin
	dd if=/dev/zero bs=512 count=100 >> ./bin/os.bin

./bin/kernel.bin: $(FILES)
	i686-elf-ld -g -relocatable $(FILES) -o ./build/kernelfull.o
	i686-elf-gcc -T ./src/linker.ld -o ./bin/kernel.bin -ffreestanding -O0 -nostdlib ./build/kernelfull.o

./bin/boot.bin: ./src/boot/boot.asm
	nasm -f bin ./src/boot/boot.asm -o ./bin/boot.bin

./build/kernel.asm.o: ./src/kernel.asm
	nasm -f elf -g ./src/kernel.asm -o ./build/kernel.asm.o

clean:
	rm -rf ./bin/boot.bin
	rm -rf ./bin/kernel.bin
	rm -rf ./bin/os.bin
	rm -rf $(FILES)
	rm -rf ./build/kernelfull.o
```
This allows us to combine our two binary outputs into object files that then become one single executable together. (boot.bin is the only non-object file in this new os.bin file we make)

## Alignment Issues:
When we start messing with C and Assembly together in the same program, we can get some alignment issues. This happens because our assembly code shares the same memory address as our compiler for C. To get around this we will be putting all our assembly code in a separate file and adding it as a section in our linker script. This will ensure our code will never be unaligned saving so many headaches in the long run.

The only issue we have with this in our current project is that we need to compile kernel.asm before everything else. This is why the Makefile is set up the way it is, to allow kernel.asm to be linked first, before our C code.
We will also add this code to the bottom of `kernel.asm`:
`times 512-($ - $$) db 0` to add to this precaution.

Our linker file should now look like this:
```ld
ENTRY(_start)
OUTPUT_FORMAT(binary)
SECTIONS
{

	. = 1M;
	.text : ALIGN(4096)
	{
		*(.text)
	}
	
	.rodata : ALIGN(4096)
	{
		*(.rodata)
	}
	
	.data : ALIGN(4096)
	{
		*(.data)
	}
	
	.bss : ALIGN(4096)
	{
		*(COMMON)
		*(.bss)
	}
	
	.asm : ALIGN(4096)
	{
		*(.asm)
	}
}
```
This now mean all assembly that we write that aren't our bootloader or kernel can specify the section for the linker to grab them by. We do this by adding `section .asm` at the top of the file. We also aligned it because it's needed for paging anyways and it will be good to get out of the way now.

## Writing C in Protected Mode:
We will be creating 2 files `kernel.c` & `kernel.h`.
```c
#include "kernel.h"

void kernel_main() {

}
```
This is our basic C file for now.
```h
#ifndef KERNEL_H
#define KERNEL_H

void kernel_main();
#endif
```
This is our basic header file for now.
```Makefile
FILES = ./build/kernel.asm.o ./build/kernel.o
INCLUDES = -I./src
FLAGS = -g -ffreestanding -falign-jumps -falign-functions -falign-labels -falign-loops -fstrength-reduce -fomit-frame-pointer -finline-functions -Wno-unused-function -fno-builtin -Werror -Wno-unused-label -Wno-cpp -Wno-unused-perameter -nostdlib -nostartfiles -nodefaultlibs -Wall -O0 -Iinc

all: ./bin/boot.bin ./bin/kernel.bin
	rm -rf ./bin/os.bin
	dd if=./bin/boot.bin >> ./bin/os.bin
	dd if=./bin/kernel.bin >> ./bin/os.bin
	dd if=/dev/zero bs=512 count=100 >> ./bin/os.bin

./bin/kernel.bin: $(FILES)
	i686-elf-ld -g -relocatable $(FILES) -o ./build/kernelfull.o
	i686-elf-gcc $(FLAGS) -T ./src/linker.ld -o ./bin/kernel.bin -ffreestanding -O0 -nostdlib ./build/kernelfull.o

./bin/boot.bin: ./src/boot/boot.asm
	nasm -f bin ./src/boot/boot.asm -o ./bin/boot.bin

./build/kernel.asm.o: ./src/kernel.asm
	nasm -f elf -g ./src/kernel.asm -o ./build/kernel.asm.o

./build/kernel.o: ./src/kernel.c
	i686-elf-gcc $(INCLUDES) $(FLAGS) -std=gnu99 -c ./src/kernel.c -o ./build/kernel.o

clean:
	rm -rf ./bin/boot.bin
	rm -rf ./bin/kernel.bin
	rm -rf ./bin/os.bin
	rm -rf $(FILES)
	rm -rf ./build/kernelfull.o
	rm -rf ./build/kernel.o
```

We can now set all our flags and make scripts to run our C code when we call build.sh.

Now we can link our C code to our kernel.asm file by doing this:
```asm
; Kernel.asm
[BITS 32]

global _start
extern kernel_main

CODE_SEG equ 0x00
DATA_SEG equ 0x10

_start:
	mov ax, DATA_SEG
	mov ds, ax
	mov es, ax
	mov fs, ax
	mov gs, ax
	mov ss, ax
	mov ebp, 0x00200000
	mov esp, ebp

; Enabling A20 Line
	in al, 0x92
	or al, 2
	out 0x92, al
	
	call kernel_main
	
	jmp $
	
times 512-($ - $$) db 0
```
This ensures that our C code is now started by our kernel. For more information go to [[Text Mode]].