# Fix: `trans_vrelu` Segmentation Fault

## Root Cause

In `target/riscv/insn_trans/trans_rvi.c.inc`, line 1542:

```c
// BUG: 0 is implicitly cast to NULL TCGv_i32 pointer
tcg_gen_brcond_i32(TCG_COND_GT, val, 0, l_if_end);
```

`tcg_gen_brcond_i32` expects `TCGv_i32` for both operands. Passing the integer literal `0` results in a NULL pointer being passed as the second operand. TCG dereferences it at runtime → segfault.

## Fix

```c
// FIXED: use tcg_constant_i32(0) to create a proper TCGv_i32 constant
tcg_gen_brcond_i32(TCG_COND_GT, val, tcg_constant_i32(0), l_if_end);
```

## Takeaway

In QEMU TCG, never pass raw integer literals (`0`, `1`, etc.) where a `TCGv_i32` / `TCGv` is expected. Always use:
- `tcg_constant_i32(val)` for `TCGv_i32`
- `tcg_constant_tl(val)` for `TCGv`
