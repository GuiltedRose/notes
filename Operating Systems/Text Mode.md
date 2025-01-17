- Allows you to write ASCII to video memory.
- Supports 16 unique colors
- No need to set individual screen pixels for printing characters.
## Text Mode ASCII:
- You write ASCII characters into memory starting at address 0xB8000 for colored displays.
- Monochrome displays address 0xB0000
- Each ASCII character written to this memory has its pixel equivalent outputted to the monitor.

## ASCII Text Mode Supported Colors:

| Color Number | Color Name   | RGB Value   | Hex Value |
| ------------ | ------------ | ----------- | --------- |
| 0            | Black        | 0 0 0       | 00 00 00  |
| 1            | Blue         | 0 0 170     | 00 00 AA  |
| 2            | Green        | 0 170 0     | 00 AA 00  |
| 3            | Cyan         | 0 170 170   | 00 AA AA  |
| 4            | Red          | 170 0 0     | AA 00 00  |
| 5            | Purple       | 170 0 170   | AA 0 AA   |
| 6            | Brown        | 170 85 0    | AA 55 00  |
| 7            | Grey         | 170 170 170 | AA AA AA  |
| 8            | Dark Grey    | 85 85 85    | 55 55 55  |
| 9            | Light Blue   | 85 85 255   | 55 55 FF  |
| 10           | Light Green  | 85 255 85   | 55 FF 55  |
| 11           | Light Cyan   | 85 255 255  | 55 FF FF  |
| 12           | Light Red    | 255 85 85   | FF 55 55  |
| 13           | Light Purple | 255 85 255  | FF 55 FF  |
| 14           | Yellow       | 255 255 85  | FF FF 55  |
| 15           | White        | 255 255 255 | FF FF FF  |
## Printing Text:
While in text mode the video card takes your ASCII value and automatically references it in a font table to output the correct pixels.

Each character takes up 2 bytes
Byte 0 = ASCII character
Byte 1 = Color code

For example we can print A.
0xb8000 = 'A'
0xb8001 = 0x00
The output will be a black "A" at row 0 column 0.

We can now print B.
0xb8002 = 'B'
0xb8003 = 0x00
The output will be a black "B" at row 0 column 1.

## Text Mode Development:
```c
// This is our header file for the kernel.
#ifndef KERNEL_H
#define KERNEL_H

#define VGA_WIDTH 80
#define VGA_HEIGHT 20

void kernel_main();
#endif
```
All we needed to add here were the defaults for VGA (Video Graphics Array). After that we can use the VGA to print text as characters, or entire strings with the code we throw into the kernel.c file.

```c
// this is the main kernel.c file.
#include "kernel.h"
#include <stdint.h>
#include <stddef.h>

  

uint16_t* video_mem = 0;
uint16_t terminal_row = 0;
uint16_t terminal_col = 0;

int16_t terminal_make_char(char c, char color) {
	return (color << 8) | c;
}

void terminal_put_char(int x, int y, char c, char color) {
	video_mem[(y * VGA_WIDTH) + x] = terminal_make_char(c, color);
}

void terminal_write_char(char c, char color) {
	if(c == '\n'){
		terminal_row += 1;	
		terminal_col = 0;
		return;
	}
	
	terminal_put_char(terminal_col, terminal_row, c, color);
	terminal_col += 1;
	if(terminal_col >= VGA_WIDTH) {
	terminal_col = 0;
	terminal_row += 1;
	}
}

void terminal_initialize() {
	video_mem = (uint16_t*)(0xB8000);
	terminal_row = 0;
	terminal_col = 0;
	for(int y = 0; y < VGA_HEIGHT; y++) {
		for(int x = 0; x < VGA_WIDTH; x++) {
			terminal_put_char(x, y, ' ', 0);
		}
	}
}

size_t strlen(const char* str) {
	size_t len = 0;
	while(str[len]) {
		len++;
	}
	
	return len;
}

void print(const char* str) {
	size_t len =strlen(str);
	for(int i = 0; i < len; i++) {
		terminal_write_char(str[i], 15);
	}
}

void kernel_main() {
	terminal_initialize();
	terminal_write_char('H', 2); //h 2
	terminal_write_char('e', 3); //e 3
	terminal_write_char('l', 13); //l 13
	terminal_write_char('l', 12); //l 12
	terminal_write_char('o', 4); //0 4
	print("\nWorld!");
}
```
There's a lot going on here, but the basics are still there. We are using `uint16_t` because it will handle a lot of the tiny steps for us. For example: if we did it the normal way we'd need to remember each hex number for each letter as well as the corresponding color code. Then we'd need to remember to write the two bytes backwards so that the processor can reorder the code correctly. When writing or put & write functions we allow the processor to handle this information for us. If you ever did any game programming it's a very basic type cast over our entire terminal interface.