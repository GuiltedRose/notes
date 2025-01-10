Some BIOS setups may corrupted our bootloader if being booted from a USB stick. 

```asm
ORG 0
BITS 16
_start:
	jmp short start
	nop

times 33 db 0

start:
	jmp 0x7c0:step2

step2:
	cli ; Clear Interrupts
	mov ax, 0x7c0
	mov ds, ax
	mov es, ax
	mov ax, 0x00
	mov ss, ax
	mov sp, 0x7c00
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

In order to ensure this we go to [[https://wiki.osdev.org/FAT]] in order to get the data we need. From here we skip 0x00's 3 bytes, and add up the rest, you should get 33.
In the above code we changed a few things, this is to ensure when the BIOS rewrites some code (if yours does) then it'll ensure that the null data gets overwritten instead(the 33 bytes of randomness). The reason it doesn't mess with our code too much is because we allow it to fill the rest of our binary file with null data until it's able to fill out the boot sector completely. This lets us put as much data in front of that code as we want as long as it doesn't exceed it.

Now we can test it by recompiling it & running QEMU.
`nasm -f bin ./boot.asm -o ./boot.bin`
`qemu-system-x86_64 -hda ./boot.bin`

![Setup for hardware](https://github.com/GuiltedRose/notes/blob/main/pictures/kernel-real-hardware-fix)
