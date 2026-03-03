# Rosemary: Binary Static Analysis Engine

**Rosemary** is a high-performance binary static analysis engine. It utilizes a custom **DSL (Domain Specific Language)** engine written in **Ruby** to define instruction sets, while leveraging a **C-based backend** for intensive semantic expansion and dataflow analysis.

---

## 1. Architecture Overview

Rosemary splits instruction definitions into two layers:
1.  **Frontend (Ruby DSL):** Defines instruction patterns, operand types, and high-level dataflow (registers).
2.  **Backend (C Engine):** Handles complex semantic expansion, memory side-effects, and flag calculations.

---

## 2. Instruction Definition (DSL)

Instructions are defined as Ruby hashes. Below is an example of an `ADD` instruction (Thumb-2/A32 style):

```ruby
{
    patterns: '001xxxxxxxxxxxxx',
    desc: 'F3.1.2.3 Add, subtract, compare, move (one low register and immediate)',
    list: [
        {
            param: '12:1,11:0', 
            mnemonic: :add,
            grammar: 'ADDS{<q>} <Rdn>, #<imm8>', 
            condition: true,
            ops_desc: [
                { t: :r_r, v: '8:3' }, # Destination/Source Register
                { t: :r_r, v: '8:3' }, # Source Register
                { t: :imm, v: '0:8' }  # 8-bit Immediate
            ],
            dataflow: { 
                def: [0], 
                use: [1], 
                out_it_def: [:n, :c, :z, :v] 
            },
            it_effect: true,
            semantic: { 
                operation: :add, 
                left: [0], 
                right: [1, 2], 
                side_effect: [:n, :c, :z, :v] 
            }
        },
    ]
}

```



### Field Definitions

- **`param`**: Specifies the bit constraints for the instruction. For example, `12:1` means the 12th bit must be **1**.
- **`mnemonic`**: The unique identifier for the instruction.
- **`grammar`**: Defines the assembly language syntax.
- **`condition`**:
  - **A32**: Indicates that bits `31..28` represent the instruction's condition code.
  - **T32**: Indicates whether the instruction can be used within an **IT (If-Then)** block.
- **`ops_desc`**: A list describing the operands of the instruction.
- **`dataflow`**: Defines the **def/use** (definition/usage) properties.
  - `def: [0]` means the first operand is defined (written).
  - `use: [1]` means the second operand is used (read).
  - Note: This field specifically tracks register dataflow; memory dataflow is handled separately.
- **`it_effect`**: If set to `true`, the assembly mnemonic becomes `adds`; otherwise, it remains `add`.

---

## 3. IR & Semantic Design

Rosemary’s IR focuses on decoupling register tracking from memory operations.

### 3.1 Memory Operands

Memory access is explicitly defined in the DSL to allow the backend to handle effective address calculations:

```ruby
{ 
  t: :mem, 
  op: MemOp::W,             # Write operation
  mode: MemMode::REG_OFFSET, # Mode: [Base + Offset]
  base_reg: :r_r, 
  v: '3:3',                  # Base register bits
  offset: { t: :r_r, v: '6:3' } # Offset register bits
}

```

### 3.2 Backend Semantic Expansion

The Ruby DSL provides the metadata, but the **C Backend** expands these into concrete operational expressions. For the `ADD` example above, the engine generates:

```c
// Intermediate Calculation
r3 = r1 + 20; // Example: imm8 (bits 0:8) resolves to 20

// Flag Logic (Context-aware)
if (out_it_block()) {
    n = NEGATIVE(r3);
    z = ZERO(r3);
    c = CARRY(r3);    // Carry flag logic
    v = OVERFLOW(r3); // Signed overflow logic
}

```
