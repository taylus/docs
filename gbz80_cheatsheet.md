# Game Boy Z80 assembly language cheatsheet

I am **terrible** at assembly. I was raised on higher-level languages, so I think in terms of variables, ifs, loops, and functions rather than registers, jumps, opcodes and the like. I put this document together as a guide to help myself remember "oh, that's how you write that in assembly." This may also help with creating macros. I didn't do that here because, like I said, I'm terrible at assembly.

<details>
<summary>Background information (click to expand)</summary>

## CPU registers and flags

* From the famous "[Pan Docs](http://gbdev.gg8.se/wiki/articles/Pan_Docs)" → http://problemkaputt.de/pandocs.htm#cpuregistersandflags

## Game Boy memory map

* http://gameboy.mongenel.com/dmg/asmmemmap.html

## GBZ80 Instruction set

* http://www.pastraiser.com/cpu/gameboy/gameboy_opcodes.html
* https://rednex.github.io/rgbds/gbz80.7.html

## Assembler toolchain

* https://github.com/rednex/rgbds

## C → GBZ80 compiler

* [Small Device C Compiler (SDCC)](http://sdcc.sourceforge.net/) helped double-check some of the examples in this guide.
* `sdcc -mgbz80 --asm=rgbds -o foo.asm foo.c`
* (seems to be targeting an older version of RGBDS?)

</details>

# "Variables"

The Game Boy has 8KB of internal work RAM and a stack you can push variables onto, but in assembly you're working with memory addresses directly. So doing something like this...

``` c
char i = 0x42
int j = 0xbeef
```

...in Z80 assembly is really just a matter of labelling addresses:

``` llvm
SECTION "global variables", WRAM0   ; RGBDS will locate these labels in work RAM bank zero
i: ds 1     ; reserve 1 byte
j: ds 2     ; reserve 2 bytes

SECTION "code", ROM0                ; RGBDS will locate these instructions in ROM bank zero
; i = 0x42
ld hl, i
ld [hl], $42

; j = 0xbeef (remember that on the Game Boy, words are 16-bit and little-endian)
ld hl, j
ld [hl], $ef
inc hl
ld [hl], $be
```

# If statements

Control flow on the Z80 is done through conditional jumps to memory addresses. An "if" statement in a higher-level language:

``` c
if (condition is true) {
    //block of code to execute only if condition is true
}
```

Is implemented as: "subtract a value from A, and based on how that affected the CPU's flags register, conditionally jump somewhere."

The `cp` instruction subtracts a value from A, doesn't store it in A, but sets the CPU's flags as if it did.

## If the `a` register == some 8-bit value (e.g. $10)

``` llvm
    cp $10  ; zero flag is set iff a == $10 because $10 - $10 == 0
    jr nz, .end_if                      ;-----+
    ; code to execute iff a == $10            |
.end_if     ;<--------------------------------+
```

* For !=, replace `jr nz` with `jr z`.

## If the `a` register < some 8-bit value (e.g. $05)

``` llvm
    cp $05  ; carry flag is set iff a < $05 because subtracting $05 from a smaller value causes a carry
    jr nc, .end_if                      ;-----+
    ; code to execute iff a < $05             |
.end_if     ;<--------------------------------+
```

* For <=, add 1 to `cp`'s operand (e.g. replace `cp $05` with `cp $06`).
* For >=, replace `jr nc` with `jr c`.
* For >, both replace `jr nc` with `jr c` and add 1 to `cp`'s operand.
* You might see `dec a` used to set the flags instead of `cp 1`, since it's more efficient if the value of `a` need not be preserved.

# If/else statements

``` c
if (condition is true) {
    //block of code to execute only if condition is true
}
else {
    //block of code to execute only if condition is false
}
```

Z80 equivalent: "Use two overlapping sets of labels and jumps, each one to skip over a block of code."

``` llvm
; (if a < $05)
    cp $05                                  ; carry flag is set iff a < $05
    jr nc, .else                            ; ----+
    ; code to execute iff a < $05                 |
    jr .end_if                              ; ----|---+
.else:                                      ; <---+   |
    ; code to execute iff a >= $05                    |
.end_if:                                    ; <-------+
```

# If/else if/else statement

``` c
if (condition is true) {
    //block of code to execute only if condition is true
}
else if (a different condition is true) {
    //block of code to execute only if a different condition is true
}
//...more else ifs...
else {
    //block of code to execute only if none of the conditions are true
}
```

Z80 equivalent: "Looks like an if/else but with even more jumps."

``` llvm
; if (a == $05)
    cp $05
    jr nz, .else_if     ;----+
    ; ...                    |
    jr .end_if          ;----|----+
.else_if:               ;<---+    |
    cp $06              ;         |
    jr nz, .else        ;---------|----+
    ; ...                         |    |
    jr .end_if          ;---------+    |
.else:                  ;<--------|----+
    ; ...                         |
.end_if:                ;<--------+
```

# Loops

``` c
while (condition is true) {
    //block of code to execute only if condition is true
}
```

Z80 equivalent: "A loop is like an if statement, you just conditionally jump backwards instead of forwards."

Here is a VRAM-clearing loop executed from the startup routine of Link's Awakening:

``` llvm
label_015d:
    ld hl, $8000            ; hl = $8000, starting address of tile pixel data in VRAM
    ld bc, $1800            ; bc = $1800, size of tile pixel data in VRAM
    call label_2999
    ...
label_2999:
    xor a                   ; a = 0;
    ldi [hl], a             ; [hl++] = 0;
    dec bc                  ; bc--;
    ld a, b                 ; a = b;
    or c                    ; a |= c;
    jr nz, label_2999       ; if ((b | c) != 0) goto label_2999;
```

Which translates to C-style loops of the form

``` c
hl = 0x8000;
bc = 0x1800;
do {
    a = 0;
    *(hl++) = a;
    bc--;
    a = b | c;
} while (a != 0);
```

or

``` c
hl = 0x8000;
bc = 0x1800;
while (bc > 0) {
    *(hl++) = 0;
    bc--;
}
```

or

``` c
for (hl = 0x8000, bc = 0x1800; bc > 0; bc--) {
    *(hl++) = 0;
}
```

# Common idioms

## xor a

Sets the register `a` to zero in half the bytes and cycles of `ld a, 0`.

How? Since a ⊕ b flips all bits in `a` where `b` is 1, a ⊕ a flips all the bits in `a` where `a` in 1, effectively setting all of its bits to zero.

## and a, or a

Check if `a` is zero in half the bytes and cycles of `cp 0`.

How? Since anything ANDed or ORed with itself equals itself, these instructions have no effect on `a`. However, they set the zero flag of the CPU if `a` is zero, which you can then use in a conditional jump.

## Clearing bits

To clear a particular bit of a value, `and` that bit position with zero. This works because either zero or one ANDed with zero results in zero.

``` llvm
; turn off the screen by clearing bit 7 of the LCD control register ($ff40)
ldh a, [$ff40]
and %01111111
ldh [$ff40], a
```

You can also use the `res` (reset) instruction:

``` llvm
; turn off the screen by clearing bit 7 of the LCD control register ($ff40)
ldh a, [$ff40]
res 7, a
ldh [$ff40], a
```

## Setting bits

Similar to the above. To set a particular bit of a value, `or` that bit position with one. This works because either zero or one ORed with one results in one. You can also use the `set` instruction to set one bit at a time.

## Flipping bits

Similar to the above. To toggle a particular bit of a value, `xor` that bit position with one. This works because either zero or one XORed with one results in the opposite of the input.

## Checking the bits of a value

The `bit` instructions are good for this, e.g.

``` llvm
bit 0, a        ; set the Z flag if bit 0 of register a is zero
```

But `and` works too, and for multiple bits at once:

``` llvm
ld b, $1
and b           ; set the Z flag if bit 0 of register a is zero
jr z, ...       ; jump somewhere if bit 0 of register a is zero

ld b, $6
and b           ; set the Z flag if bits 1 and 2 of register a are zero
jr nz ...       ; jump somewhere if bits 1 and 2 of register a are not zero
```

## Registers/naming conventions

Aside from register `a` being the accumulator and `hl` being used for indirect addressing because certain instructions only work with those registers, some of the general purpose registers are commonly used for particular purposes.

At first I thought these were just cute naming conventions, but after looking into the Z80 more, it's apparently from some of the instructions that were dropped from the Game Boy's version of the processor (e.g. [LDI](http://z80-heaven.wikidot.com/instructions-set:ldi)).

| Register  | Name/purpose                                                          |
| --------- |-----------------------------------------------------------------------|
| de        | "Destination address" for memory copying routines.                    |
| bc        | "Byte count," the number of bytes to copy in memory copying routines. |
