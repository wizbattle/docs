# VM Design

The game uses a *Virtual Machine* on the server side to handle spells
in combat. This chapter explains the design of the VM and how it works.

## Motivation

This approach brings several advantages with it, which outweigh the
other considered options:

- **Data, not code:** Spells are decoupled from the implementation and
  can be modified and added without needing to recompile the server.

- **Sandboxed:** Malicious or buggy spells can't bring the game down
  while still allowing for easy debugging.

- **Compact:** Bytecode images of spells are significantly smaller in
  size than their C++ object representations.

- **Stable:** We can ensure a stable data format to store even between
  changes to the game's C++ types between revisions.

  This is also attractive for relational databases because OOP class
  hierarchies don't lend themselves to nice database schemas.

- **Efficient:** Spells have minimal resource footprint, which keeps
  the server manageable even under heavy load.

- **Accurate:** A spell's bytecode can be disassembled and turned into
  an accurate description of effects for human players. This may help
  with familiarizing with new game mechanics when they're released.

## Registers

The wizbattle Virtual Machine is register-based. It supports reading
and writing values to them and do maths in-between.

We feature a total of 16 registers, 4 of them for general-purpose
computations and 12 for special-purpose setup.

The special-purpose registers are zeroed after every spell. Code generators
may rely on this behavior and don't have to do manual cleanup.

### GPRs

The 4 GPRs are labeled `0` through `3` and behave identical. Use them as
you wish with no drawbacks.

### ZERO

A 32-bit special-purpose register that always reads `0` and ignores writes,
labeled `4`.

### ARG

A 32-bit signed integer argument to the spell, labeled `5`.

Most of the time, this will be the base damage or base healing amount of
a spell. In other cases, this may not be used at all.

Spells are expected to load this value in on demand.

### TGT

Encodes the target of the spell, labeled `6`.

Below is an illustration of the register's bitfield:

```
 31                                7  6   5  4   0
+-----------------------------------+---+---+-----+
|                                   |   |   |     |
|                RESERVED           | C |DIV| SEL |
|                                   |   |   |     |
+-----------------------------------+---+---+-----+
```

**SEL**: The effect target according to `enum SpellEffect::kEffectTarget`.

**DIV**: Whether damage should be divided between the targets.

**C**: Whether critical hit rolls are disabled for this spell.

### AP

A 32-bit signed integer argument to encode additional armor piercing,
labeled `7`.

### Reserved

The registers `8` through `15` are currently not in use and reserved
for future extensions.

## Instructions

The VM executes a linear stream of instructions (commands), which modify
its internal state and carry out the logic to modify the game state.

### Arithmetic

> **MOV $dst, $src**: Moves a value from the source register to the destination register.

> **LOAD $dst, off**: Loads a 32-bit constant into $dst from byte offset `off << 1`.

> **ADD $dst, $src**: Adds the source register to the destination register.

> **MUL $dst, $src**: Multiplies the destination register with the source register.

> **DIV $dst, $src**: Divides the destination register by the source register.

> **AND $dst, $src**: Bitwise AND of destination register and source register.

> **OR $dst, $src**: Bitwise OR of destination register and source register.

> **XOR $dst, $src**: Bitwise XOR of destination register and source register.

> **BSET $dst, idx**: Sets a bit of a given index in the destination register.

### Branch

> **JMP off**: Unconditionally jumps over `off << 1` code bytes.

> **J.cond $reg1, $reg2, off**: Conditionally jumps over `off << 1` code bytes. The
  condition code `cond` decides how the two input registers are compared.

### External

> **CMD.ATTACK**: Triggers an attack with the setup of special-purpose registers.
  Overwrites the base damage in `ARG` with the actual damage dealt.

> **CMD.HEAL**: Triggers healing with the setup of special-purpose registers.

> **CMD.DRAIN**: Heals for `ARG` HP without factoring any additional boosts
  to incoming and outgoing healing in.

> **CMD.RESHUFFLE**: Reshuffles the spell deck.

> **CMD.HALT**: Halts execution of the current bytecode image. Hitting the end of
  the bytecode image without encountering this instruction is an error.
