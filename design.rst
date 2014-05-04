CPU Design Notes
================

Instruction Format
------------------

Instructions are two bytes long::

    CCCC CTTT LLLL RRRR
           or AAAAAAAAA

    C: opcode
    T: target
    L: left operand
    R: right operand
    A: memory address operand

Load, store and branching instructions use the 8-bit address form. Arithmetic and logical instructions
use the 4-bit left/right operand form.

In spite of its name, "target" may sometimes be a source operand when the 8-bit address form is used.

The "target" is one bit smaller than the two operands because in operands the MSB is used to
signal an immediate value, but immediate targets are not permitted. See the "operand format" 
section below for more details.

Opcodes
-------

The most significant bit of the opcode is set for all operations that use the ALU, with the remaining
three bits acting as the ALU's opcode. Other opcodes do not interact with the ALU.

The arithmetic operations (ADD, ADC, SUB, SUC) all perform two's complement arithmetic.

The following symbols are used to describe the operations::

     T: Target
     L: Left Operand
     R: Right Operand
     A: Memory Address Operand
    PC: Program Counter
    SP: Stack Pointer
     *: Dereference ("The memory address given in...")

=====  ========  ============================================================
Code   Mnemonic  Description
=====  ========  ============================================================
00000  CAL       push PC to SP, PC = A
00001  CAI       push PC to SP, PC = *T
00010  RET       PC = pop 
00011  REI       PC = pop SP, enable interrupts
00100  STO       *A = T
00101  STI       *T = R (L ignored)
00110  LOA       T = *A
00111  LOI       T = *R (L ignored)
01000  JMC       PC = A if (T & 1) == 1
01001  JMP       PC = A
01010  JMI       PC = *T
01011  NOP       no operation
01100  BRC       PC = PC + R if (T & 1) == 1 (L ignored)
01101  BRA       PC = PC + R (T, L ignored)
01110  IEN       enable interrupts
01111  IDI       disable interrupts
10000  ADD       T = L + R
10001  ADC       T = L + R + carry
10010  SUB       T = L - R
10011  SBC       T = L - R - ~carry
10100  AND       T = L & R
10101  OR        T = L | R
10110  XOR       T = L ^ R
10111  EQ        T = ~(L ^ R)
11000  SHL       T = L << R
11001  SHR       T = L >> R
11010  NE        T = (L == R) ? 0 : 1
11011  EQ        T = (L == R) ? 1 : 0
11100  LT        T = (L < R) ? 1 : 0
11101  GT        T = (L > R) ? 1 : 0
11110  
11111  
=====  ========  ============================================================

Notes:

* The ADD/ADC and SUB/SUC pairs intentionally differ only by the LSB so that we can simply AND
  the LSB with the carry flag to get the carry in for the adder/subtractor.

Operand Format
--------------

The T, L and R parts of an instruction all use a common encoding:

====  ========================================
Code  Meaning
====  ========================================
1XXX  Immediate 3-bit number
0000  Program Counter
0001  Register 1
0010  Register 2
0011  Register 3
0100  Register 4
0111  Stack Pointer
====  ========================================

Other operand codes with MSB=0 are reserved for future expansion.

The T part of an instruction lacks the 4th bit and thus cannot represent immediate values.

Immediate value operands are sign-extended to 8 bits, causing them to be interpreted as
twos-complement values. Thus immediate values have a range from -4 to 3 inclusive.
Due to the limited range, immediate values are best suited to incrementing or decrementing
loop induction variables.

Since immediate values are *always* sign-extended, care must be taken when using them with
bitwise operations. For example, "x & 1" would have the desired effect of masking out
everything except the low-order bit, but "x | 0b100" would be expanded as "x | 0b1111100",
not "x | 0b0000100" as one might expect.

For some opcodes the "target" is specified as an input. In this case its value drives the
L bus multiplexer instead of T as normal, and the T bus multiplexers are disabled. Such opcodes
may therefore not use the "L" portion of the instruction.

Buses
-----

The design includes five separate buses, which can be interconnected for certain operations:

* Data Bus: Input/output of data to/from memory devices. (RAM, ROM)
* Address Bus: Input of address to memory devices. (RAM, ROM)
* L Bus: Value of the L operand in the instruction. (immediate value or register value)
* R Bus: Value of the R operand in the instruction. (immediate value or register value)
* T Bus: Result value of the instruction.

Interconnects
-------------

L and R Buses
^^^^^^^^^^^^^

Inputs:

* Immediate value (sign-extended 3 LSBs from IR)

* R1, R2, R3, R4 and SP registers

* Program Counter

Outputs:

* ALU

* Address Bus

T Bus
^^^^^

Inputs:

* ALU (for arithmetic operations)

* Data Bus (for 'load' operations)

Outputs:

* R1, R2, R3, R4 and SP registers

* Program Counter (LSB ignored and fixed at 0 to ensure an even number)

Data Bus
^^^^^^^^

Inputs:

* Memory Data (when reading memory)

Outputs:

* Memory Data (when writing memory)

* IR (when fetching an instruction)

* T Bus (when loading data from memory into a register)

Address Bus
^^^^^^^^^^^

Inputs:

* PC (when fetching an instruction)

* Address Value (8 LSBs from IR)

* L and R buses (when handing an indirect memory access)

Outputs:

* Memory Address (permanently connected)

Component Notes
---------------

Atmel AT28C256: 256k (32k * 8) Paged Parallel EEPROM
http://www.digikey.com/product-detail/en/AT28C256-15PU/AT28C256-15PU-ND/1008506

Texus Instruments SN74LS181N: 4-bit ALU
http://www.digikey.com/product-detail/en/SN74LS181N/296-33973-5-ND/1594771

Renesas R1LP0108ESP-5SI#B0: 1M Parallel SRAM (32-SOP)
http://www.digikey.com/product-detail/en/R1LP0108ESP-5SI%23B0/R1LP0108ESP-5SI%23B0-ND/2694359

Texas Instruments SN74LS593N: 8-bit counter
http://www.digikey.com/product-detail/en/SN74LS593N/296-3719-5-ND/377750

Texas Instruments SN74HC574N: 8-bit D Flip-flop
http://www.digikey.com/product-detail/en/SN74HC574N/296-1598-5-ND/277244

Texas Instruments 74HCT4051N,112: 8-to-1 Multiplexer
http://www.digikey.com/product-detail/en/74HCT4051N,112/568-7851-5-ND/1230893


