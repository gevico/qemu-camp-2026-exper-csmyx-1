# trans_crush Debugging & Fix Summary

  ## Debugging trans_sort with GDB

  ### Setup
  - Build with debug: `../configure --target-list=riscv64-linux-user --enable-debug && make -j$(nproc)`
  - Use the existing test infrastructure: `make -C build/tests/gevico/tcg/riscv64-softmmu gdbstub-insn-sort`
    - This launches QEMU with `-s -S` (gdbserver on `:1234`, halted)
  - Connect from a second terminal: `riscv64-unknown-elf-gdb build/tests/gevico/tcg/riscv64-softmmu/test-insn-sort`

  ```gdb
  target remote :1234
  break trans_sort
  continue
  print *a    # inspect rs1, rs2, rd

  gdb-multiarch vs riscv64-unknown-elf-gdb

  - gdb-multiarch can misdetect architecture and apply wrong register layouts (e.g. x86 FPU on RISC-V), causing assertion errors like i387_supply_fxsave
  - Use riscv64-unknown-elf-gdb for softmmu targets — built specifically for RISC-V, no ambiguity

  TCG IR logging (no GDB needed)

  QEMU_LOG=op ./build/qemu-system-riscv64 -M g233 -m 2G -display none -semihosting \
    -device loader,file=test-insn-crush -d op -D /tmp/crush_tcg.log

  ---
  trans_crush Compile Errors

  Root cause

  All address variables (src_addr, dst_addr, etc.) were declared as TCGv_i32, but tcg_gen_qemu_ld/st_i32 expects TCGv (= TCGv_i64 on riscv64) for the address
  argument.

  Fix

  - Declare address variables as TCGv, keep value variables as TCGv_i32
  - Use tcg_gen_extrl_i64_i32(N, get_gpr(...)) to truncate a TCGv register into a TCGv_i32
  - Use tcg_gen_ext_i32_tl(offset, val) to widen a TCGv_i32 into a TCGv for address arithmetic

  ---
  trans_crush Segmentation Fault

  Diagnosis

  QEMU itself crashed (not the guest), so the GDB remote connection closed immediately. Used TCG op log to inspect generated IR.

  Key observations from /tmp/crush_tcg.log:
  - brcond_i32 loc16,loc16,eq,$L3 — val was reused for both the load result and the N & 0x1 odd-check, so it compared with itself
  - Mixed TCGv_i32 / TCGv (i64) temps in brcond_i32 loop condition caused backend register allocation issues

  Fix: make everything consistently TCGv

  static bool trans_crush(DisasContext *ctx, arg_crush *a)
  {
      MemOp mop = MO_8;
      TCGv N = tcg_temp_new();
      TCGv src_base = tcg_temp_new();
      TCGv dst_base = tcg_temp_new();
      TCGv src_addr = tcg_temp_new();
      TCGv dst_addr = tcg_temp_new();
      TCGv out_len = tcg_temp_new();
      TCGv i = tcg_temp_new();
      TCGv i_bound = tcg_temp_new();
      TCGv_i32 val = tcg_temp_new_i32();
      TCGv_i32 val_1 = tcg_temp_new_i32();

      tcg_gen_mov_tl(N, get_gpr(ctx, a->rs2, EXT_NONE));
      tcg_gen_mov_tl(src_base, get_gpr(ctx, a->rs1, EXT_NONE));
      tcg_gen_mov_tl(dst_base, get_gpr(ctx, a->rd, EXT_NONE));
      tcg_gen_mov_tl(src_addr, src_base);
      tcg_gen_mov_tl(dst_addr, dst_base);
      tcg_gen_addi_tl(out_len, N, 1);
      tcg_gen_shri_tl(out_len, out_len, 1);
      tcg_gen_shri_tl(i_bound, N, 1);

      TCGLabel *l_i_loop = gen_new_label();
      TCGLabel *l_i_end  = gen_new_label();
      tcg_gen_movi_tl(i, 0);

      gen_set_label(l_i_loop);
      tcg_gen_brcond_tl(TCG_COND_GE, i, i_bound, l_i_end);

      tcg_gen_qemu_ld_i32(val, src_addr, ctx->mem_idx, mop);
      tcg_gen_addi_tl(src_addr, src_addr, 1);
      tcg_gen_andi_i32(val, val, 0x0f);
      tcg_gen_qemu_ld_i32(val_1, src_addr, ctx->mem_idx, mop);
      tcg_gen_addi_tl(src_addr, src_addr, 1);
      tcg_gen_andi_i32(val_1, val_1, 0x0f);
      tcg_gen_shli_i32(val_1, val_1, 4);
      tcg_gen_or_i32(val, val_1, val);
      tcg_gen_qemu_st_i32(val, dst_addr, ctx->mem_idx, mop);
      tcg_gen_addi_tl(dst_addr, dst_addr, 1);

      tcg_gen_addi_tl(i, i, 1);
      tcg_gen_br(l_i_loop);
      gen_set_label(l_i_end);

      TCGLabel *l_if_end = gen_new_label();
      TCGv odd = tcg_temp_new();
      tcg_gen_andi_tl(odd, N, 0x1);
      tcg_gen_brcond_tl(TCG_COND_EQ, odd, tcg_constant_tl(0), l_if_end);
      /* src_addr already at src_base+N after loop */
      tcg_gen_qemu_ld_i32(val, src_addr, ctx->mem_idx, mop);
      tcg_gen_andi_i32(val, val, 0x0f);
      tcg_gen_mov_tl(dst_addr, dst_base);
      tcg_gen_addi_tl(dst_addr, dst_addr, -1);
      tcg_gen_add_tl(dst_addr, dst_addr, out_len);
      tcg_gen_qemu_st_i32(val, dst_addr, ctx->mem_idx, mop);
      gen_set_label(l_if_end);

      return true;
  }

  ---
  Key Takeaways

  ┌───────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────┐
  │               Rule                │                                          Detail                                          │
  ├───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
  │ Address args must be TCGv         │ tcg_gen_qemu_ld/st_i32(val, addr, ...) — addr is always TCGv regardless of value width   │
  ├───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
  │ Don't mix i32/i64 in loop control │ Use brcond_tl / addi_tl consistently; mixing causes backend register allocation failures │
  ├───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
  │ Don't reuse value temps for flags │ Using val for both a load result and a bitmask check causes self-comparison bugs         │
  ├───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
  │ Odd-element tail simplification   │ After the loop, src_addr already points to src_base + N; no need to recompute offset     │
  ├───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
  │ Use tcg_constant_tl(0)            │ For immediate comparisons in brcond_tl instead of a literal 0                            │
  ├───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
  │ ```                               │                                                                                          │
  └───────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────┘