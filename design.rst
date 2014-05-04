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
0001  Register 1
0002  Register 2
0003  Register 3
0004  Register 4
0100  Program Counter
0101  Stack Pointer
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

Component Notes
---------------

Atmel AT28C256: 256k (32k * 8) Paged Parallel EEPROM
http://www.digikey.com/product-detail/en/AT28C256-15PU/AT28C256-15PU-ND/1008506

Texus Instruments SN74LS181N: 4-bit ALU
http://www.digikey.com/product-detail/en/SN74LS181N/296-33973-5-ND/1594771



