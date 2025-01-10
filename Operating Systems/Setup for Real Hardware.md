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

![Setup for hardware](https://github.com/GuiltedRose/notes/blob/main/pictures/kernel-real-hardware-fix.png?raw=true)

## Writing to the USB Drive:
`lsblk` -> (list block) this will list all the drives on your operating system.
The reason I like using lsblk over fdisk (format disk) or similar commands is the fact you can see what's on the device before writing directly to it. It's easier to ensure your system is safe.
![lsblk output](https://github.com/GuiltedRose/notes/blob/main/pictures/lsblk-output.png?raw=true)
I decided to show you both with and without the USB so you can see more clearly what's being added onto the system. In my case I'd format sdb so it doesn't affect my system.

The next command we want to be super careful with which is why we used lsblk to ensure we are 100% correct in what data is being overwritten.
`cd [OS Directory]` -> We want to switch to the folder our OS project is located in.
`sudo dd if=./boot.bin of=/dev/[Device ID]` -> we then use dd(disc destroyer) to overwrite the USB with our operating system's bootloader. To understand what the command is doing let's take a closer look:
* dd -> disc destroyer
* if=./boot.bin-> input file . means any folder in current directory that has the file 'boot.bin'
* of =/dev/sdb-> output file /dev directory manipulating the file sdb (yes, all drives are considered files).
If it doesn't work you should do some research on your profile to find a manual specific to the architecture of that processor. It's not an end-all-be-all script, it's just a test.
[[Interrupt Vector Table]]
