## Files Do Not Exist!
1. Filesystems are kernel implemented; they are not the responsibility of the hard disk (This is why there are so many types).
2. Implementing a filesystem requires the kernel developer to create a filesystem driver for the target filesystem.

Data is read in written in sectors typically 512 byte blocks.
* CHS(Cylinder, Head, Sector) -> How HDDs write data.
	sectors are read and written by specifying a "head", "track", and "sector".
	This isn't really used a whole lot anymore, since most modern drives are Solid State and not Hard Disk.
* LBA(Logical Block Address) -> How SSDs write data.
	This is the more modern way of reading from a hard disk, instead of "head", "track", "sector"  we just specify a number that starts from 0.
	LBA allows us to read from the disk as if we are reading blocks from a very large file.
	LBA 0 = first sector on the disk
	LBA 1 = second sector on the disk
	etc.

## Calculating LBA:
Let's say we want to read byte at position ==58376== on the disk, how do we do this?

LBA = 58376 / 512 -> we divide our position by the number of bytes per sector.
LBA = 114.015625
Now if we read that LBA, we can load 512 bytes into memory.
Next we need to know our offset that our byte is in our buffer.
Offset = 58476 % 512 = 8 -> we use the modulus operator to get the offset.
The math:
LBA = 114 (we floor it because of the offset).
Offset = 8
114 * 512 = 58,368
58,368 + 8 = ==58,376==

In 16-bit real mode the BIOS provides interrupt 13h for disk operations.
In 32 & 64-bit modes you ***MUST*** create your own disk drivers which is more complicated than the BIOS handing them to us.

## Reading from Hard Disk:
We will start by creating a Makefile. If you haven't made one before, don't worry it's just an instruction set that tells our assembler & compiler what to do. **NOTE: the spelling matters! Makefile**

```Makefile
all:
	nasm -f bin ./boot.asm -o ./boot.bin
	dd if=./message.txt >> ./boot.bin
```
This will let us build our binary file much faster, and will become more useful later on in the protected mode section to come.

Now we can create a text file with whatever text you wish in order to test the code we're about to write.

I saved mine as `message.txt` with `This is a message that does message stuff.` as the contents.
we now need to install `make` & `bless` to do the rest of the steps.
Now if we run bless with the contents of `boot.bin`, we should see this:
![bless output](https://github.com/GuiltedRose/notes/blob/main/pictures/bless-output.png?raw=true)
If you take a good look our message should appear at the start of the second sector.

```Makefile
all:
	nasm -f bin ./boot.asm -o ./boot.bin
	dd if=./message.txt >> ./boot.bin
	dd if=/dev/zero bs=512 count=1 >> ./boot.bin
```
We now want to add a block after our message so when we manipulate our code to read it to the screen, it will actually do so. Before this it couldn't be accessed because it's appended after the boot sector ends.
![bless output updated](https://github.com/GuiltedRose/notes/blob/main/pictures/bless-updated-output.png?raw=true)
If done correctly you should see extra data appended to the end of our message.
**NOTE: the BIOS sets the DL register to the hard disk number by default**
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

	mov ah, 2 ; READ SECTOR COMMAND
	mov al, 1 ; NUMBER OF SECTORS TO READ
	mov ch, 0 ; Cylinder low eight bits
	mov cl, 2 ; sector numbers start at 1.
	mov dh, 0 ; Head number
	mov bx, buffer
	int 0x13
	jc error
	jmp $
	
error:
	mov si, error_message
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

error_message: db 'Failed to load sector', 0

times 510-($ - $$) db 0
dw 0xAA55

buffer: ; 0x7E00
```
So to explain what's going on here, we need to start our block at 2 in order for the hard disk interrupt to be used. (See [This](http://www.ctyme.com/intr/rb-0607.htm)for more details). 
We start at 2 for the read sector command, then we tell it the number of sectors to read, which is 1. Next we pick which cylinder to read which is the first in our case 0. Then we tell it we want the data from sector 2 & tell it that we want header 0. After that we set our buffer to just after our boot sector. ==**NOTE: this can only be referenced as a buffer, we cannot write data here physically!**==
We also set up some error handling in the form of a message to signal to us if something went wrong. If all went well we should see this:
![blank output](https://github.com/GuiltedRose/notes/blob/main/pictures/blank.png?raw=true)
This is because we never told our program to write the stored message like we did for the error code.

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
	
	mov ah, 2 ; READ SECTOR COMMAND
	mov al, 1 ; NUMBER OF SECTORS TO READ
	mov ch, 0 ; Cylinder low eight bits
	mov cl, 2 ; sector numbers start at 1.
	mov dh, 0 ; Head number
	mov bx, buffer
	int 0x13
	jc error
	
	mov si, buffer
	call print
	
	jmp $
	
error:
	mov si, error_message
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

error_message: db 'Failed to load sector', 0

times 510-($ - $$) db 0
dw 0xAA55

buffer: ; 0x7E00
```
Now we should be able to see our manipulated text appear!
![File Data Shown](https://github.com/GuiltedRose/notes/blob/main/pictures/file-input-boot.png?raw=true)
This works because text files are null terminated, we can change the data as much as we want and it'll change the output each time.
