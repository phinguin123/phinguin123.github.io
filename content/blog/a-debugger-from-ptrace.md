---
title: "Writing a Tiny Debugger with ptrace"
date: 2023-07-23T12:00:00+09:00
draft: false
categories: ["Systems"]
tags: ["c", "linux", "ptrace", "x86-64", "debugging", "low-level"]
summary: "A small C debugger that single-steps a program, rewrites its registers and memory, and changes what it does. Then figuring out why real debuggers don't work like that — a breakpoint is literally one byte, 0xCC."
---

Back in 2023 I wrote a small debugger in C on top of `ptrace`. This was one of the assignments I had to do for a specific course! (Can't say which one)

It ran a program one instruction at a time, counted the instructions inside `main`, jumped over two blocks of code by rewriting the instruction pointer, and wrote a string straight into a stack buffer. It worked. But it also never used a single breakpoint, which I only really understood way later — it stopped the program 268,886 times to act on 2 addresses. With actual breakpoints the same thing takes 3 stops.

I didn't write anything down back then and I don't have the original target binary anymore. So for this post I rebuilt the same setup on a small test program. The debugger code is my original one, but the addresses, disassembly and numbers come from the test program.

## So what is a debugger anyway

Nothing special, basically. On Linux it's just a normal process that the kernel lets do four things to another process:

1. Stop it, and get told whenever it stops.
2. Read/write its registers, including `rip` (address of the next instruction).
3. Read/write its memory, including its code.
4. Resume it, either freely or for exactly one instruction.

All four come from one syscall, `ptrace`. gdb uses the same one.

## Getting control before the program runs

```c
pid_t pid = fork();
if (pid == 0) {
    ptrace(PTRACE_TRACEME, 0, NULL, NULL);   /* "my parent may trace me" */
    execl(argv[1], argv[1], NULL);           /* become the target program */
} else {
    debugger(pid);
}
```

Child asks to be traced, then turns itself into the target with `execl`. Since it's being traced, the kernel stops it with a `SIGTRAP` right after `execve` succeeds — before `main`, even before the dynamic loader runs. So the parent's first `wait()` returns with the target loaded and frozen, and from there the parent decides everything.

## One instruction at a time

My main loop:

```c
wait(&wait_status);
while (WIFSTOPPED(wait_status)) {
    ptrace(PTRACE_GETREGS, pid, 0, &regs);
    if (regs.rip >= MAIN_START && regs.rip <= MAIN_END) {
        inst_count++;
        /* ... inspect or change things ... */
    }
    ptrace(PTRACE_SINGLESTEP, pid, NULL, NULL);
    wait(&wait_status);
}
printf("[+] total instruction : %u\n", inst_count);
```

`PTRACE_SINGLESTEP` is actually a CPU feature. On x86 the kernel sets the trap flag (TF) in `RFLAGS` and resumes the target. CPU runs exactly one instruction, raises a debug exception, kernel turns that into another `SIGTRAP`, target stops again. `wait()` returns, I read the registers, repeat until the program exits (`WIFSTOPPED` goes false).

### Why hardcoded addresses even worked

The address range in that `if` was copied straight from the disassembly. That only works because the target was built without PIE. A non-PIE binary always loads at `0x400000`, so the code sits at the same addresses every run. ASLR still moves the stack and shared libs, just not the executable.

`main` from the test program:

```text
4011b4:  endbr64
4011b8:  push   rbp
4011b9:  mov    rbp,rsp
4011bc:  sub    rsp,0x70
4011c0:  call   401196 <check>
   ...
4011d8:  mov    rdx,QWORD PTR [rip+0x2e51]    # stdin
4011df:  lea    rax,[rbp-0x70]                # the buffer
4011e3:  mov    esi,0x64
4011e8:  mov    rdi,rax
4011eb:  call   4010a0 <fgets@plt>
4011f0:  lea    rax,[rbp-0x70]
   ...
```

With PIE you'd have to read the real base from `/proc/<pid>/maps` after the target stops and add offsets to it.

## Making the program do something else

The target is frozen before every instruction, so changing `rip` decides what runs next. A jump the program never asked for:

```c
if (regs.rip == SKIP_FROM) {
    regs.rip = SKIP_TO;                        /* jump over a block */
    ptrace(PTRACE_SETREGS, pid, 0, &regs);
}
```

Used this twice. First one skips a block entirely (in the test program it's a call to `check()` plus the branch on its result — honestly don't remember what it was in the original). Second one skips the code that fills a stack buffer, and I write my own string into the buffer instead. The disassembly tells you where the buffer is: `lea rax,[rbp-0x70]` right before the call means it starts at `rbp - 0x70`.

### POKEDATA writes a word, not a byte

My first try wrote the string in 8-byte chunks:

```c
unsigned long long value_to_write = 0x2f2f3a7370747468;
ptrace(PTRACE_POKEDATA, pid, regs.rbp - 0x70, (void *)value_to_write);
```

That number is `"https://"`. x86 is little-endian so the lowest byte goes to the lowest address — read `0x2f2f3a7370747468` from the right and you get `68 74 74 70 73 3a 2f 2f`, `h t t p s : / /`. Problem was the 2nd and 4th chunks had wrong offsets, I subtracted 8 instead of adding it... so part of the string went *below* the buffer into other stack memory.

So I switched to a loop that writes one char at a time:

```c
for (size_t i = 0; i < data_length; i++)
    ptrace(PTRACE_POKEDATA, pid, regs.rbp - 0x70 + i, (void *)data_to_write[i]);
```

Works, but not for the reason it looks like. `PTRACE_POKEDATA` always writes a full 8-byte word. Every call writes the char plus seven zero bytes, and the next call one byte higher overwrites six of those zeros. To see what's left over, I filled the buffer with `X` (`0x58`) first and ran the loop with a 21-char string:

```text
[rbp-0x70+ 0] 7266206f6c6c6568
[rbp-0x70+ 8] 7420656874206d6f
[rbp-0x70+16] 0000007265636172
[rbp-0x70+24] 5858585800000000   <- seven X's turned into zeros
```

The last write left seven zero bytes past the end. Here that kinda helped since it null-terminated the string for free, but it's still writing past the end. If the string had almost filled the buffer, those bytes would've hit whatever's next on the stack — another variable or the saved frame pointer.

Proper way is read-modify-write. Read the word, change only the bytes you want, write it back:

```c
static void write_bytes(pid_t pid, unsigned long addr, const char *src, size_t n) {
    for (size_t i = 0; i < n; i += 8) {
        unsigned long w = ptrace(PTRACE_PEEKDATA, pid, addr + i, 0);
        size_t k = n - i < 8 ? n - i : 8;
        memcpy(&w, src + i, k);
        ptrace(PTRACE_POKEDATA, pid, addr + i, (void *)w);
    }
}
```

## I never set a single breakpoint

Everything above did what I wanted. It was also the slowest possible way to do it lol.

| On the test program        | Stops   | Time     |
| -------------------------- | ------- | -------- |
| Single-step everything     | 268,886 | ~15 s    |
| Instructions inside `main` | 19      |          |
| Breakpoints (below)        | 3       | ~0.004 s |

Single-stepping stops on every instruction from the very first one — dynamic loader, libc startup, `fgets`, `printf`, the exit path. Each stop is two context switches and a bunch of syscalls. Out of 268,886 stops I cared about 2 addresses. Kind of like watching a movie frame by frame to find one scene instead of just skipping to it.

## A breakpoint is one byte

x86 has a one-byte instruction, `int3` (`0xCC`), whose whole job is to raise a breakpoint exception. To set a breakpoint, a debugger:

1. Saves the original byte at that address.
2. Overwrites it with `0xCC`.
3. Lets the program run normally with `PTRACE_CONT`.

(`ptrace` can write into code even though code pages are read-only — the kernel makes a private copy of the page for that process.)

When execution gets there, the CPU runs `int3` instead of the real instruction, kernel turns it into `SIGTRAP`, debugger's `waitpid` returns. One thing to watch: `int3` already executed, so `rip` is one byte *past* the breakpoint. That `- 1` is how you know which breakpoint got hit:

```c
static void set_bp(pid_t pid, struct bp *b) {
    b->orig = ptrace(PTRACE_PEEKTEXT, pid, b->addr, 0);
    ptrace(PTRACE_POKETEXT, pid, b->addr, (void *)((b->orig & ~0xffUL) | 0xcc));
}
```

```c
ptrace(PTRACE_GETREGS, pid, 0, &r);
unsigned long hit = r.rip - 1;    /* int3 already ran: rip is one byte past it */
```

### Continuing past one

If you just want to look and keep going, there's a problem — the first byte of that instruction is still `0xCC`. So:

1. Put the original byte back.
2. Rewind `rip` by one.
3. Single-step that one instruction.
4. Write `0xCC` back in and continue.

```c
clear_bp(pid, &bp);                       /* restore the real instruction */
r.rip = hit;                              /* rewind onto it */
ptrace(PTRACE_SETREGS, pid, 0, &r);
ptrace(PTRACE_SINGLESTEP, pid, 0, 0);     /* run just that one instruction */
waitpid(pid, &st, 0);
set_bp(pid, &bp);                         /* re-arm */
ptrace(PTRACE_CONT, pid, 0, 0);
```

So single-stepping doesn't disappear, it just shrinks down to the one instruction under each breakpoint. My two "skip" breakpoints are even easier since they move `rip` somewhere else and the original instruction never runs.

### Hardware breakpoints

x86 also has debug registers: `DR0`–`DR3` hold up to four addresses and `DR7` turns them on. The CPU checks every instruction fetch against them, so no code gets modified (useful if the program checks its own bytes). They can also fire on memory access instead of execution, which is what a watchpoint is ("stop when anything writes to this variable") — software breakpoints can't do that. On Linux you set them through `PTRACE_POKEUSER` at the offset of `u_debugreg` in `struct user`. Only four though.

## Same thing, with breakpoints

Skip a block, replace the buffer contents, plus one normal "look and continue" breakpoint:

```c
int main(int argc, char *argv[]) {
    pid_t pid = fork();
    if (pid == 0) { ptrace(PTRACE_TRACEME, 0, 0, 0); execl(argv[1], argv[1], NULL); _exit(127); }

    int st; struct user_regs_struct r;
    struct bp skip_check = { 0x4011c0 }, skip_fgets = { 0x4011d8 }, at_printf = { 0x401221 };
    const char s[] = "hello from the tracer";

    waitpid(pid, &st, 0);                           /* stopped at exec */
    set_bp(pid, &skip_check); set_bp(pid, &skip_fgets); set_bp(pid, &at_printf);
    ptrace(PTRACE_CONT, pid, 0, 0);

    while (waitpid(pid, &st, 0), WIFSTOPPED(st)) {
        ptrace(PTRACE_GETREGS, pid, 0, &r);
        unsigned long hit = r.rip - 1;
        if (hit == skip_check.addr) {
            r.rip = 0x4011d8;                       /* jump over the block */
        } else if (hit == skip_fgets.addr) {
            write_bytes(pid, r.rbp - 0x70, s, sizeof s);   /* includes the '\0' */
            r.rip = 0x4011f0;                       /* skip the fgets call */
        } else if (hit == at_printf.addr) {
            fprintf(stderr, "[bp] printf about to run, rsi=%#llx\n", r.rsi);
            /* restore, rewind, step, re-arm (as above) */
            continue;
        }
        ptrace(PTRACE_SETREGS, pid, 0, &r);
        ptrace(PTRACE_CONT, pid, 0, 0);
    }
}
```

```text
[bp] printf about to run, rsi=0x7ffed2c95f00
buf = hello from the tracer
[+] stops: 3
```

Three stops, full speed in between.

## Other stuff I'd fix in the 2023 version

- Signals were getting swallowed. Passing `0` as the last arg to `PTRACE_SINGLESTEP`/`PTRACE_CONT` cancels whatever signal stopped the target. Fine for my own `SIGTRAP`s, but if the program got a real `SIGINT` or `SIGSEGV`, my debugger just ate it. Should pass `WSTOPSIG(status)` back when it's not a trap.
- `PEEKDATA` returns a `long` and I stored it in an `unsigned int`, so the top half was gone. Also `-1` is both a valid word and the error value, so real error checking means clearing `errno` before and checking after.
- Used `wait()` instead of `waitpid(pid, ...)`. Doesn't matter with one child, but still.
- Didn't handle a failed `execl`. Wrong path and the child just keeps running the rest of `main`. `_exit(127)` right after fixes that.

The assignment taught me the stuff gdb hides from you — a debugger is just a process the kernel lets mess with another one, and changing `rip` is a jump the program never wrote. What it didn't teach me is that single-stepping is for tracing everything, not for stopping at a couple of places. For that you just write `0xCC`.
