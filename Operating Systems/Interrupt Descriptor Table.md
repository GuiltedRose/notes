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
There's no numbers because they're incremented in order; just like the IVT. For example Offset 16-31 would be the first offset in this list, while "Int 5" is "Gate Type".