# Implementing `trans_expand` for QEMU RISC-V TCG Frontend

## Instruction Semantics

`expand` is the inverse of `crush`. It unpacks N packed bytes (each holding two 4-bit nibbles) into 2*N bytes with one nibble per byte.

- `rs1` = src base address
- `rd`  = dst base address  
- `rs2` = N (number of packed source bytes)
- `dst[2*i]   = src[i] & 0x0F`  (low nibble)
- `dst[2*i+1] = (src[i] >> 4) & 0x0F`  (high nibble)

## Changes Made

### `target/riscv/insn32.decode`
- Uncommented `@r_expand` format and `expand` instruction entry:
```
@r_expand  .......   ..... ..... ... ..... ....... %rs2 %rs1 %rd
expand     0110110 ..... ..... 110 ..... 1111011 @r_expand
```

### `target/riscv/insn_trans/trans_rvi.c.inc`
Implemented `trans_expand`:
```c
static bool trans_expand(DisasContext *ctx, arg_expand *a)
{
    MemOp mop = MO_8;
    TCGv N        = tcg_temp_new();
    TCGv src_addr = tcg_temp_new();
    TCGv dst_addr = tcg_temp_new();
    TCGv i        = tcg_temp_new();
    TCGv_i32 val  = tcg_temp_new_i32();
    TCGv_i32 lo   = tcg_temp_new_i32();
    TCGv_i32 hi   = tcg_temp_new_i32();

    tcg_gen_mov_tl(N,        get_gpr(ctx, a->rs2, EXT_NONE));
    tcg_gen_mov_tl(src_addr, get_gpr(ctx, a->rs1, EXT_NONE));
    tcg_gen_mov_tl(dst_addr, get_gpr(ctx, a->rd,  EXT_NONE));
    tcg_gen_movi_tl(i, 0);

    TCGLabel *l_loop = gen_new_label();
    TCGLabel *l_end  = gen_new_label();

    gen_set_label(l_loop);
    tcg_gen_brcond_tl(TCG_COND_GE, i, N, l_end);

    tcg_gen_qemu_ld_i32(val, src_addr, ctx->mem_idx, mop);
    tcg_gen_addi_tl(src_addr, src_addr, 1);

    tcg_gen_andi_i32(lo, val, 0x0f);
    tcg_gen_shri_i32(hi, val, 4);
    tcg_gen_andi_i32(hi, hi, 0x0f);

    tcg_gen_qemu_st_i32(lo, dst_addr, ctx->mem_idx, mop);
    tcg_gen_addi_tl(dst_addr, dst_addr, 1);
    tcg_gen_qemu_st_i32(hi, dst_addr, ctx->mem_idx, mop);
    tcg_gen_addi_tl(dst_addr, dst_addr, 1);

    tcg_gen_addi_tl(i, i, 1);
    tcg_gen_br(l_loop);
    gen_set_label(l_end);

    return true;
}
```

## Troubleshooting

**Issue:** First test run showed all-zero output (`hw=0x00 sw=0x0b`).  
**Cause:** `qemu-system-riscv64` had not been rebuilt after editing the source.  
**Fix:** Run `ninja qemu-system-riscv64` in the `build/` directory before running the test.

## Build & Test Commands

```sh
# Rebuild QEMU
cd build && ninja qemu-system-riscv64

# Run test
make -C build/tests/gevico/tcg/riscv64-softmmu/ run-insn-expand
```
