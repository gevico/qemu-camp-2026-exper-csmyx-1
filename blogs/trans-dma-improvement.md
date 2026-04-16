# Improving `trans_dma` in QEMU TCG

## What `dma` does

The `dma` custom instruction transposes an N×N matrix of `float32` elements:

```
dst[j*N + i] = src[i*N + j]
```

`N = 8 << grain`, so grain=0 → 8×8, grain=1 → 16×16, grain=2 → 32×32.

Registers: `rd` = dst base, `rs1` = src base, `rs2` = grain.

---

## Problems with the original implementation

### 1. Unsafe use of `get_gpr` for `rd`

```c
TCGv dst_base = get_gpr(ctx, a->rd, EXT_NONE);
```

`get_gpr` returns a reference to the live register TCGv, not a copy. Using it as
`dst_base` inside a loop that emits many TCG ops is unsafe — the value can be aliased
or clobbered. Base pointers must be snapshotted into fresh temps before the loop.

### 2. Two multiplies per inner iteration

```c
tcg_gen_mul_tl(src_addr, i, N);
tcg_gen_add_tl(src_addr, src_addr, j);
tcg_gen_muli_tl(src_addr, src_addr, 4);   // * 4
tcg_gen_add_tl(src_addr, src_addr, src_base);

tcg_gen_mul_tl(dst_addr, j, N);
tcg_gen_add_tl(dst_addr, dst_addr, i);
tcg_gen_muli_tl(dst_addr, dst_addr, 4);   // * 4
tcg_gen_add_tl(dst_addr, dst_addr, dst_base);
```

Every inner iteration recomputes `i*N+j` and `j*N+i` from scratch using multiplies.
For a 32×32 matrix that's 1024 × 4 multiply ops emitted into the TB.

### 3. Wrong memory op type

```c
tcg_gen_qemu_ld_i32(val, src_addr, ctx->mem_idx, MO_32);
tcg_gen_qemu_st_i32(val, dst_addr, ctx->mem_idx, MO_32);
```

`MO_32` doesn't include endianness. Should use `MO_UL | mo_endian(ctx)` and operate
on `TCGv` (target-word-sized) rather than `TCGv_i32`.

---

## Improved implementation

The key insight: instead of recomputing addresses with multiplies each iteration,
maintain two running pointers that increment by a fixed stride:

- `src_ptr` advances by `4` each j step → sequential read along row i
- `dst_ptr` advances by `stride` (N×4) each j step → column-stride write into dst
- After each i step: `src_row += stride`, `dst_col += 4`

```c
static bool trans_dma(DisasContext *ctx, arg_dma *a)
{
    MemOp mop = MO_UL | mo_endian(ctx);

    TCGv N        = tcg_temp_new();
    TCGv stride   = tcg_temp_new();
    TCGv src_base = tcg_temp_new();
    TCGv dst_base = tcg_temp_new();
    TCGv i        = tcg_temp_new();
    TCGv src_row  = tcg_temp_new();
    TCGv dst_col  = tcg_temp_new();
    TCGv j        = tcg_temp_new();
    TCGv src_ptr  = tcg_temp_new();
    TCGv dst_ptr  = tcg_temp_new();
    TCGv val      = tcg_temp_new();

    tcg_gen_movi_tl(N, 8);
    tcg_gen_shl_tl(N, N, get_gpr(ctx, a->rs2, EXT_NONE));
    tcg_gen_shli_tl(stride, N, 2);  /* stride = N * 4 */

    /* snapshot base pointers */
    tcg_gen_mov_tl(src_base, get_gpr(ctx, a->rs1, EXT_NONE));
    tcg_gen_mov_tl(dst_base, get_gpr(ctx, a->rd,  EXT_NONE));

    TCGLabel *l_i_loop = gen_new_label();
    TCGLabel *l_i_end  = gen_new_label();
    tcg_gen_movi_tl(i, 0);
    tcg_gen_mov_tl(src_row, src_base);
    tcg_gen_mov_tl(dst_col, dst_base);

    gen_set_label(l_i_loop);
    tcg_gen_brcond_tl(TCG_COND_GE, i, N, l_i_end);

    TCGLabel *l_j_loop = gen_new_label();
    TCGLabel *l_j_end  = gen_new_label();
    tcg_gen_movi_tl(j, 0);
    tcg_gen_mov_tl(src_ptr, src_row);
    tcg_gen_mov_tl(dst_ptr, dst_col);

    gen_set_label(l_j_loop);
    tcg_gen_brcond_tl(TCG_COND_GE, j, N, l_j_end);

    tcg_gen_qemu_ld_tl(val, src_ptr, ctx->mem_idx, mop);
    tcg_gen_qemu_st_tl(val, dst_ptr, ctx->mem_idx, mop);

    tcg_gen_addi_tl(src_ptr, src_ptr, 4);
    tcg_gen_add_tl(dst_ptr, dst_ptr, stride);
    tcg_gen_addi_tl(j, j, 1);
    tcg_gen_br(l_j_loop);
    gen_set_label(l_j_end);

    tcg_gen_add_tl(src_row, src_row, stride);
    tcg_gen_addi_tl(dst_col, dst_col, 4);
    tcg_gen_addi_tl(i, i, 1);
    tcg_gen_br(l_i_loop);
    gen_set_label(l_i_end);

    return true;
}
```

---

## Summary of changes

| | Original | Improved |
|---|---|---|
| Base pointer safety | `get_gpr` reference (unsafe) | snapshotted into fresh `TCGv` temps |
| Address computation | 2× `mul_tl` + 2× `muli_tl` per inner iter | pointer increment only (`addi` / `add`) |
| Stride computation | `muli_tl(x, x, 4)` each iter | `shli_tl(stride, N, 2)` once before loop |
| Memory op | `qemu_ld/st_i32` with `MO_32` | `qemu_ld/st_tl` with `MO_UL \| mo_endian(ctx)` |
