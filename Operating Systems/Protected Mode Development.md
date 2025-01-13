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
This is needed in order to access the 21st byte in any memory source, without it this byte is completely inaccessible and means we have 0 keyboard support from what I have read on it. Obviously for an operating system this is just as bad as not being able to boot.

## Setting Up Cross Compiler:
In order to do this we can go to [OS Dev](https://wiki.osdev.org/GCC_Cross-Compiler). Once there, we can download all the tools it tells us we need to build our own cross compiler. The reason we are doing this is because our current compiler is already linked to our native operating system (it should be linux for a much easier experience). After that we need the source files for `build-essentials` & `gcc` both are developed by the GNU foundation. Once downloaded we will extract them to our home folder (because we are going to use this OS eventually it shouldn't really matter too much). They are also super easy to clean up after we're done with them. 