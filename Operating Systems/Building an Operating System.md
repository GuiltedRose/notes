## Tools needed for "Real Mode":
*  NASM & GCC - C compiler & asm assembler
* qemu-system-x86 - emulator for kernels
To run qemu: `qemu-system-x86_64`
* [Interrupt Codes](https://www.ctyme.com/intr/int.html)
* [OS Development Resources](https://wiki.osdev.org/Expanded_Main_Page)
* [Encryption Documentation](https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.197.pdf)(Will be moved later).
## Memory:
Hardware that allows a computer to store information.

RAM -> Random Access Memory:
* A volatile memory type that clears after a reboot.
* This is the main memory of a computer that allows for reading and writing of information.
* Used anytime you need to manipulate data such as a programming variable.
ROM -> Read-Only Memory:
* Non-volatile memory that persists after a reboot.
* Can only be read from, think a CD ROM.
* Cannot be written to through normal means.
* Used a lot for embedded devices like a raspberry pi.
*  BIOS is written to ROM.

# The Boot Process:
The BIOS is executed from ROM -> A bootloader is loaded from the BIOS -> our bootloader loads the kernel to start the OS.

## What is a bootloader?
* A program responsible for loading the kernel of an operating system.
* They are generally small.
* These load us into "Protected Mode" which will be discussed later.
* Generally written by the OS developer.
## What is the BIOS?
* Executed directly from ROM.
* The CPU executes instructions directly from the BIOS' ROM.
* The BIOS generally loads itself into RAM then continues execution from RAM.
* The BIOS will initialize essential hardware.
* The BIOS looks for a bootloader to boot by searching all storage mediums for the boot signature "0x55AA".
* The BIOS loads the bootloader into RAM at absolute address 0x7c00.
* The BIOS instructs the process to preform a jump to absolute address "0x7c00" and begin executing the OS' bootloader.
* The BIOS contains routines to assist our bootloader in booting our kernel.
* The BIOS is 16 bit so it can only execute 16 bit code.
* The BIOS routines are generic & standard will be expanded upon later.
