# Implementing `trans_vdot` for QEMU RISC-V TCG Frontend

## Instruction Semantics

`vdot rd, rs1, rs2` — INT32 vector dot product of two 16-element arrays.

- `rs1` = base address of array A (int32[16])
- `rs2` = base address of array B (int32[16])
- `rd`  = result (int64 accumulator)
- `rd = sum_{i=0}^{15} (int64)A[i] * (int64)B[i]`

Encoding: `.insn r 0x7b, 6, 70` → funct7=`1000110`, funct3=6, opcode=`1111011`

## Changes Made

### `target/riscv/insn32.decode`
Uncommented format and instruction entry:
```
@r_vdot    .......   ..... ..... ... ..... ....... %rs2 %rs1 %rd
vdot       1000110 ..... ..... 110 ..... 1111011 @r_vdot
```

### `target/riscv/insn_trans/trans_rvi.c.inc`
```c
static bool trans_vdot(DisasContext *ctx, arg_vdot *a)
{
    MemOp mop = MO_SL | mo_endian(ctx);
    TCGv a_addr = tcg_temp_new();
    TCGv b_addr = tcg_temp_new();
    TCGv i      = tcg_temp_new();
    TCGv acc    = tcg_temp_new();
    TCGv_i32 va = tcg_temp_new_i32();
    TCGv_i32 vb = tcg_temp_new_i32();
    TCGv ea     = tcg_temp_new();
    TCGv eb     = tcg_temp_new();
    TCGv prod   = tcg_temp_new();

    tcg_gen_mov_tl(a_addr, get_gpr(ctx, a->rs1, EXT_NONE));
    tcg_gen_mov_tl(b_addr, get_gpr(ctx, a->rs2, EXT_NONE));
    tcg_gen_movi_tl(acc, 0);
    tcg_gen_movi_tl(i, 0);

    TCGLabel *l_loop = gen_new_label();
    TCGLabel *l_end  = gen_new_label();
    TCGv n = tcg_constant_tl(16);

    gen_set_label(l_loop);
    tcg_gen_brcond_tl(TCG_COND_GE, i, n, l_end);

    tcg_gen_qemu_ld_i32(va, a_addr, ctx->mem_idx, mop);
    tcg_gen_qemu_ld_i32(vb, b_addr, ctx->mem_idx, mop);

    tcg_gen_ext_i32_tl(ea, va);   /* sign-extend to 64-bit */
    tcg_gen_ext_i32_tl(eb, vb);
    tcg_gen_mul_tl(prod, ea, eb);
    tcg_gen_add_tl(acc, acc, prod);

    tcg_gen_addi_tl(a_addr, a_addr, 4);
    tcg_gen_addi_tl(b_addr, b_addr, 4);
    tcg_gen_addi_tl(i, i, 1);
    tcg_gen_br(l_loop);
    gen_set_label(l_end);

    gen_set_gpr(ctx, a->rd, acc);
    return true;
}
```

## Key Points

- Elements are `int32` loaded with `MO_SL` (sign-extending load) and then sign-extended to `TCGv` (64-bit) via `tcg_gen_ext_i32_tl` before multiply — avoids overflow.
- Result is written back with `gen_set_gpr`, not a store — `rd` is a register, not a memory address.
- `tcg_constant_tl(16)` used for the loop bound instead of a temp, since it's a compile-time constant.

## Build & Test

```sh
ninja qemu-system-riscv64
make -C build/tests/gevico/tcg/riscv64-softmmu/ run-insn-vdot
```
