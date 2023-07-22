# Scsh IR Spec

This document outlines the specification for the intermediate code used for a program written in `Scsh`.
The intermediate code uses one-byte mnemonics to represent various operations and uses 3 Address Codes (3AC).

Each operand can be either a source `src` or destination `dest`.

## Format of a Bytecode File
The bytecode file follows the following format:

```
FILE TYPE         : 4 B : string "scsh" ]
VERSION           : 2 B                 ]
CONST POOL SIZE   : 8 B                 ]
CODE SECTION SIZE : 8 B                 ] <-- file headers (38 B)
EXTERN REF SIZE   : 8 B                 ]
ENTRY POINT ADDR  : 8 B                 ]
CONST POOL        : variable size
CODE SECTION      : variable size
EXTERN REF        : variable size
```

The bytecode file has an address to the code's entry point after 30 bytes.

### Versions
Version code is split into 1 byte major and 1 byte minor codes. Changes to major indicates breaking changes.

An exception is that major version `0` i.e. all `0.x` indicates unstable releases and each upgrade should be considered a breaking change.

### Constant Pool
The constant pool is the data section of the bytecode file that stores read only data.

The constant pool starts after 38 bytes from the start of the bytecode file and extends until the code section.

The section after the const pool is the code section.

Data is organised sequentially without any padding (except for null term in strings), with the address acting as marker for the starting of a data block.

A data block has the following format:
```
DATA TYPE ID    : 1 B
DATA            : variable size
```

The data type ID tells the runtime how far the data actually goes

As an exception, for lists the following format is used:
```
DATA TYPE ID : 1 B
LENGTH       : 8 B
DATA TYPE ID : 1 B
ELEMENT      : variable size
DATA TYPE ID : 1 B
ELEMENT      : variable size
DATA TYPE ID : 1 B
ELEMENT      : variable size
.
.
.
```

### Data Type IDs
```
enum DataType_t {
    DATA_TYPE_BUL = 0,        /* boolean        : 1 B */
    DATA_TYPE_CHR = 1,        /* char           : 1 B */
    DATA_TYPE_I64 = 2,        /* int64_t        : 8 B */
    DATA_TYPE_F64 = 3,        /* double         : 8 B */
    DATA_TYPE_STR = 4,        /* char*          : variable  */
    DATA_TYPE_INTERP_STR = 5, /* parsable char* : variable  */
    DATA_TYPE_LST = 6,        /* list           : variable  */
    DATA_TYPE_ANY = 7,        /* void*          : undefined */
};
```

### External Functions
This section comes after the code section and is a list of names of `libc` functions (or externally linked C functions if support for that is ever added).

When the execution label tells the runtime to jump into this section, the runtime uses the execution label to extract out the function name and make a call to it.

The runtime calculates where it is currently reading from by using the various size information given in the header.

## Runtime Features
The following are the features of the runtime itself.

### Variable Table
This is a stack of variable maps such that on every `CALL`, a new map (scope) is pushed to the stack.

This map contains only the variables from the current function call. Once `RET` is executed, the top map is popped and freed from memory.

Each variable is accessible using a 4 B address starting from 32.

The first 32 addresses (0 - 31) are reserved as a static global memory.

### Global Addresses
A static and global global 32 * 8 byte array memory is used for function call arguments and temporary variables.
These addresses are accessed from 0 - 31.

These are technically not stored in the variable table even though they are accessed using the addressing mode corresponding to the variable table.

However, they can store whatever the variable table can store.

These addresses are cleared every time a procedure returns.

## Instructions
### Addressing Modes
A `src` or source operand in the intermediate code can be either constant or not constant.

If it's a constant, it technically is an address to the constant pool.
If it's not a constant, it is an address to the variable table.

```
src1 is constant : 0------- ]
src1 is address  : 1------- ]
src2 is constant : -0------ ]<-- 1 byte opcodes
src2 is address  : -1------ ]
```

### Operands
- `src`: A `4 B` address to the variable table or the constant pool, determined by the addressing mode.
- `dest`: A `4 B` address to the variable table only.
- `lbl`: An `8 B` address pointing to the code section.

### Mnemonics
- In case a corresponding `src` field in the opcode is `1`, it indicates for that opcode that `src` operand is always an address.
- `X` indicates don't care i.e. that field is ignored (for example if the corresponding operand doesn't exist).
- `Y` indicates a possible valid mode (0 or 1)
- `_` seperates the addresing mode from the code.

#### Table Format:
```
- TOKEN                        : INSTRUCTION           : OPCODE
```

#### Unary Operators:
```
- LEXTOK_INCREMENT             : INC src               : 0b1X_000000
- LEXTOK_DECREMENT             : DEC src               : 0b1X_000001
- LEXTOK_MINUS                 : NEG src               : 0b1X_000010
```

#### Arithmetic Operators:
```
- LEXTOK_PLUS                  : ADD src1, src2, dest  : 0bYY_000011
- LEXTOK_MINUS                 : SUB src1, src2, dest  : 0bYY_000100
- LEXTOK_ASTERIX               : MUL src1, src2, dest  : 0bYY_000101
- LEXTOK_FSLASH                : DIV src1, src2, dest  : 0bYY_000110
- LEXTOK_PERCENT               : MOD src1, src2, dest  : 0bYY_000111
```

#### Logical Operators:
```
- LEXTOK_LOGICAL_AND           : AND src1, src2, dest  : 0bYY_001000
- LEXTOK_LOGICAL_OR            : OR  src1, src2, dest  : 0bYY_001001
- LEXTOK_BANG                  : NOT src, dest         : 0bYY_001010
```

#### Bitwise Operators:
```
- LEXTOK_AMPERSAND             : BAND src1, src2, dest : 0bYY_001011
- LEXTOK_PIPE                  : BOR  src1, src2, dest : 0bYY_001100
- LEXTOK_CARET                 : XOR  src1, src2, dest : 0bYY_001101
- LEXTOK_TILDE                 : INV  src, dest        : 0bYY_001110
- LEXTOK_BITWISE_LSHIFT        : LSH  src1, src2, dest : 0bYY_001111
- LEXTOK_BITWISE_RSHIFT        : RSH  src1, src2, dest : 0bYY_010000
- LEXTOK_ARITH_RSHIFT          : ASR  src1, src2, dest : 0bYY_010001
```

#### Relational Operators:
```
- LEXTOK_LBRACE_ANGULAR        : LT src1, src2, dest   : 0bYY_010010
- LEXTOK_RBRACE_ANGULAR        : GT src1, src2, dest   : 0bYY_010011
- LEXTOK_LOGICAL_LESSER_EQUAL  : LE src1, src2, dest   : 0bYY_010100
- LEXTOK_LOGICAL_GREATER_EQUAL : GE src1, src2, dest   : 0bYY_010101
- LEXTOK_LOGICAL_EQUAL         : EQ src1, src2, dest   : 0bYY_010110
```

#### Control Codes:
```
- /* branch if zero       */   : BRZ  src, lbl         : 0bYX_010111
- /* branch if +ve        */   : BRP  src, lbl         : 0bYX_011000
- /* branch if -ve        */   : BRN  src, lbl         : 0bYX_011001
- /* unconditional branch */   : BR   lbl              : 0bYX_011010
- /* function call        */   : CALL lbl              : 0bYX_011011
- LEXTOK_KWD_RETURN            : RET  src              : 0bYX_011100
```

#### Data Move Code:
```
- LEXTOK_ASSIGN                : MOV src, dest         : 0bYX_011101
```

## Notes
- The `YY` part of the mnemonic gives the addressing modes of `src1` and `src2` respectively.
- The remaining 6 bits provide 64 combinations for mnemonics.
- So, important operations should be given mnemonics, while others can be derived from them.
- Endianess is managed by converting the code using `htonl` when saving the bytecode.
- The runtime uses `ntohl` to get the data out.

## Sample
The following sample is written in pseudo assembly style and calculates the factorial of a number.
```
.start main

.const
    ip_str: string     "Enter a number: "
    op_str: interp_str "result = {1}\n"
    one   : i64        1
    two   : i64        2

.code

; factorial(num: [1]) -> [0]
factorial:
    SUB [1], $one, [32]
    BRZ [32], .recurse
    BRN [32], .recurse
    RET $one
    .recurse:
        MOV [1], [33]
        SUB [1], $one, [1]
        CALL factorial
        MOV [33], [1]
        MUL [1], [0], [0]
        RET [0]

main:
    MOV $ip_str, [1]
    MOV $two, [2]
    CALL input
    MOV [0], [1]
    CALL factorial
    MOV [0], [2]
    MOV $op_str, [1]
    CALL print
```
