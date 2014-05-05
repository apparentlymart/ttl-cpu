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

ALU is permanently connected to the L and R buses. The Address Bus
may be optionally connected to one of these buses at a time via its
input mux.

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

* L bus

* Memory

Outputs:

* IR (when fetching an instruction) (Low/High Bytes are two separate outputs)

* T Bus (when loading data from memory into a register)

NB: Bridging the L bus to the Data Bus implies placing the memory into write mode, which
effectively turns the memory into an output. This is used during the STO and STI operations,
which operate with the L bus populated from the register selected by T.

Address Bus
^^^^^^^^^^^

Inputs:

* PC (when fetching an instruction)

* Address Value (8 LSBs from IR)

* L and R buses (when handing an indirect memory access)

Outputs:

* Memory Address (permanently connected)

Control Word
------------

The following control signals are included in a control word:

====  =================================================================================
Bits  Meaning
====  =================================================================================
1     PC will increment on clock pulse
1     T Bus Active (clock pulse reaches selected register)
1     Memory in Write Mode, L Bus bridged to Data Bus, L Bus populated from T selector
1     ALU enabled (will assert on T bus)
2     Data Bus Output Selection (None, IR L, IR H, T Bus)
2     Addr Bus Input Selection (PC, L Bus, R Bus, Addr)
====  =================================================================================

* L/R buses are always active with the value selected by the
  corresponding operand, but for two exceptions: when the address
  bus input is addr (so we're interpreting the IR LSB as an
  address rather than two operands), and when the memory is
  in write mode due to the exception noted below.

* When L/R buses are active the L and R instruction operands are
  decoded and the corresponding value asserted on them.

* When the T bus is active, the T instruction operand is decoded,
  and the clock signal and T bus outputs are connected to the
  selected register.

* Something is always asserting on the address bus, but it
  can be effectively disconnected by disabling either the L or R
  bus and selecting that bus.

* The ALU is implicitly activated when the MSB of the opcode
  in the instruction register is 1 AND the L and R buses are active.
  When it is active, it asserts on the T bus. When it is inactive
  it is disconnected. The ALU is always deactivated for a non-ALU
  instruction.

* The T bus input is automatically selected based on the current
  instruction: if it's an ALU instruction than the ALU is selected.
  Otherwise, the data bus bridged to the T bus.

* Whenever the L/R buses are active they are *always* connected to
  the ALU. Addr Bus Input Selection allows one of the buses to
  additionally be bridged into the address bus for indirect writes.

* As hinted at in the table, the memory being in write mode has two
  special side-effects: the L bus is populated based on the T selector
  rather than the L selector, and the L bus is bridged into the
  data bus. This is used to execute both STO and STI, in which the
  value to be stored is read from a register given in the T selector,
  whereas for all other instructions T is a register to write *to*.
  R remains populated as normal in this mode however, unless the
  address bus input is also selecting "addr".

* When Data Bus Output is selecting either IR L or IR H, the clock
  is automatically connected to the selected register so it will
  update from the Data Bus at the next clock pulse.

* If the data bus output is set to either IR L or IR H *and* memory
  is in write mode, the value of the register selected by T would
  be written into one of the IRs. This is not a valid state.

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


