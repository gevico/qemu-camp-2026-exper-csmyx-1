# Fix: `make -f Makefile.camp test-cpu` fails with `error: InvalidCharacter`

## Symptom

Running `make -f Makefile.camp test-cpu` fails at `test-insn-dma` even though the test
passes when run manually:

```
TEST      1/10   test-insn-dma on riscv64
make[2]: *** [.../Makefile.softmmu-target:43: run-insn-dma] Error 1
```

The test log at `build/tests/gevico/tcg/riscv64-softmmu/test-insn-dma.log` contains:

```
error: InvalidCharacter
```

## Investigation

The test runner macro in `tests/gevico/tcg/Makefile.target` wraps every test with:

```makefile
run-test = ... timeout -s KILL --foreground $4 $2 > $1.log 2>&1 ...
```

Running the QEMU command directly works fine (exit 0), but wrapping it with `timeout`
fails (exit 1):

```bash
# works
/path/to/qemu-system-riscv64 -M g233 ... -device loader,file=test-insn-dma
# exit: 0, output: "dma: all tests passed"

# fails
timeout -s KILL --foreground 120 /path/to/qemu-system-riscv64 ...
# exit: 1, output: "error: InvalidCharacter"
```

Checking which `timeout` is actually being used:

```bash
$ which timeout
/home/hugo/local/zigcli-v0.3.0-x86_64-linux/bin/timeout

$ timeout --version
Usage:
 timeout SECONDS COMMAND [ARG]...
```

The zigcli `timeout` does not support `-s KILL` or `--foreground` flags and errors out
instead of running the command. It shadows the GNU coreutils `timeout` at `/usr/bin/timeout`.

## Root Cause

`/home/hugo/local/zigcli-v0.3.0-x86_64-linux/bin` appears before `/usr/bin` in `$PATH`,
so the wrong `timeout` binary is picked up.

## Fix

Prepend `/usr/bin` to `PATH` when running the tests:

```bash
PATH=/usr/bin:$PATH make -f Makefile.camp test-cpu
```

To fix permanently, reorder `$PATH` in `~/.zshrc` so `/usr/bin` comes before the zigcli
directory:

```bash
# ~/.zshrc — move /usr/bin before zigcli
export PATH=/usr/bin:$PATH:/home/hugo/local/zigcli-v0.3.0-x86_64-linux/bin
```

Or simply remove the zigcli directory from `PATH` if `timeout` from it is not needed.
