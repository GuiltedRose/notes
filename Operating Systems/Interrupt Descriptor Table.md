* Describes how interrupts are invoked in protected mode.
* Can be mapped anywhere in memory.
* Different from the interrupt vector table.
* Similarly to the interrupt vector table; the interrupt descriptor table describes how interrupts are setup in the CPU so that if someone causes an "int 5" it will invoke the code for interrupt 5 as described by the IDT.

## Interrupt Descriptor:

| Name     | Bit   | Known As                   | Description                                                           |
| -------- | ----- | -------------------------- | --------------------------------------------------------------------- |
| offset   | 46-63 | Offset 16-31               | The higher part of the offset to execute.                             |
| P        | 47    | Present                    | This should be set to zero for unused interrupts.                     |
| DPL      | 45-46 | Descriptor Privilege Level | The ring level the processor requires to call this interrupt.         |
| S        | 44    | Storage Segment            | Should be set to zero for trap gates.                                 |
| Type     | 40-43 | Gate Type                  | The type of gate this interrupt is treated as.                        |
| 0        | 32-39 | Unused 0-7                 | Unused bits in this structure.                                        |
| Selector | 16-31 | Selector 0-15              | The selector this interrupt is bound to I.e the kernel code selector. |
| Offset   | 0-15  | Offset 0-15                | The lowest part of the offset to execute.                             |
There's no numbers because they're incremented in order; just like the IVT. This is just a list of bits we need to set for certain behaviors depending on the interrupt we are invoking, and who can actually invoke the interrupt.
Here's a C example:
```c
struct idt_desc {
	uint16_t offset_1; // offset bits 0..15
	uint16_t selector; // a code segment selector in GDT or LDT
	uint8_t zero; // unused, set to 0
	uint8_t type_attr; // type and atributes, see below
	uint16_t offset_2; // offset bits 16..32
} __attribute__((packed)); // helps for structure alignment
```

## Interrupt Gate Types:

| Name                        | Value       | Description                                                                                                                                                   |
| --------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 80386 32-bit Task Gate      | 0x05/0b0101 | Tasks gates reference TSS descriptors and can assist in multi-tasking when exceptions occur.                                                                  |
| 80286 16-bit Interrupt Gate | 0x06/0b0110 | Interrupts gates are to be used for interrupts that we want to invoke ourselves in our code.                                                                  |
| 80386 16-bit Trap Gate      | 0x07/0b0111 | Trap gates are like interrupt gates however, they are used for exceptions. They also disable interrupts on entry and re-enable them on an 'iret" instruction. |
| 80386 32-bit Interrupt Gate | 0x0E/0b1110 | Interrupts gates are to be used for interrupts that we want to invoke ourselves in our code.                                                                  |
| 80386 32-bit Trap Gate      | 0x0F/0b1111 | Trap gates are like interrupt gates however, they are used for exceptions. They also disable interrupts on entry and re-enable them on an 'iret" instruction. |
## Interrupt Descriptor Array

| Interrupt 0 descriptor | Interrupt 1 descriptor | Interrupt 3 descriptor | ... |
| ---------------------- | ---------------------- | ---------------------- | :-: |

```c
struct idt_desc idt_desc[COS32_MAX_INTERRUPTS];
```
Interrupt descriptors are stored in an array with index 0 defining interrupt zero "int 0". Index 1 defining interrupt one "int 1" and so on.

## IDTR

| Name  | Bit   | Description                                             |
| ----- | ----- | ------------------------------------------------------- |
| Limit | 0-15  | The length of the Interrupt Descriptor Table minus one. |
| Base  | 16-47 | The address of the Interrupt Descriptor Table.          |
```c
struct idtr_desc {
	uint16_t limit;
	uint32_t base;
} __attribute__((packed));
```

Loading interrupt descriptor table:
```asm
idt_load:
	push ebp
	mov ebp, esp
	mov ebx, [ebp+8]
	lidt [ebx]
	pop ebp
	ret
```
We can define where we want the IDT in memory.
IDTs are setup differently than IVTs.
During an interrupt certain properties can be pushed to the stack. The rules involved with this are quite complex so we will discuss them as they come and they do not always apply.

## IDT Development:
We create a new folder in our `src` folder called `idt`. This is where all the IDT code will reside.
We will then create our first files as `idt.h`, `idt.c`, and `idt.asm`. While we're at it we also need to make a `memory` folder in our `src` directory as well. This will house our `memory.h`, and `memory.c` files. We should also make these folders in the `build` directory (needed for our Makefile later).

```c
#ifndef IDT_H
#define IDT_H
#include <stdint.h>

struct idt_desc {
	uint16_t offset_1; // Offset bits 0 - 15
	uint16_t selector; // Set to GDT selector
	uint8_t zero; // unused; does nothing
	uint8_t type_attr; // Descriptor type
	uint16_t offset_2; // Offset bits 16 - 32
} __attribute__((packed));

  

struct idtr_desc {
	uint16_t limit; // Size of IDT - 1
	uint32_t base; // Base address of the start of the table.
} __attribute__((packed));

void idt_init();

#endif
```
This is where we set our logic for our pointer functions we will be making later.

```asm
section .asm
global idt_load

idt_load:
push ebp
mov ebp, esp

mov ebx, [ebp+8]
lidt [ebx]
pop ebp
ret
```
This will let us call `idt_load` in our `idt.c` code.
```c
#include "idt.h"
#include "config.h"
#include "kernel.h"
#include "memory/memory.h"

struct idt_desc idt_descriptors[KYRIX_TOTAL_INTERRUPTS];
struct idtr_desc idtr_descriptor;

extern void idt_load(struct idtr_desc* ptr);

void idt_zero() {
	print("Divide by zero error\n");
}

void idt_set(int interrupt_no, void* address) {
	struct idt_desc* desc = &idt_descriptors[interrupt_no];
	desc->offset_1 = (uint32_t) address & 0x0000ffff;
	desc->selector = KERNEL_CODE_SELECTOR;
	desc->zero = 0x00;
	desc->type_attr = 0xEE;
	desc->offset_2 = (uint32_t) address >> 16;
}

void idt_init() {
	memset(idt_descriptors, 0, sizeof(idt_descriptors));
	idtr_descriptor.limit = sizeof(idt_descriptors) -1;
	idtr_descriptor.base = (uint32_t) idt_descriptors;
	
	idt_set(0, idt_zero);
	// Load IDT
	idt_load(&idtr_descriptor);
}
```
This is the main function for our IDT That we can import into `kernel.c`.