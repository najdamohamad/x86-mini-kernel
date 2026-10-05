# Mini x86 Kernel: Console, Timer Interrupts & Process Scheduling

A small 32-bit x86 kernel written from scratch in C and assembly. It drives the VGA text screen, handles the hardware timer interrupt and switches between kernel processes. It runs in QEMU.

![Language](https://img.shields.io/badge/language-C%20%2F%20x86%20assembly-blue)

> Academic project — Ensimag (Grenoble INP), 2021 — individual project.

## Overview

The kernel boots under QEMU with no operating system underneath, so every feature works directly on the hardware: video memory, I/O ports, the interrupt table and the CPU registers. The project was built in three stages:

1. **Screen output**: a console driver so that `printf` works inside the kernel
2. **Time**: programming the hardware timer, handling its interrupt and showing a live clock
3. **Processes**: process table, context switching and timer-driven round-robin scheduling

## What I implemented

### Console driver (`start.c`)
- Writes characters directly to VGA text memory at `0xB8000` (80×25 cells, each holding a character byte and a color byte)
- Reads and moves the hardware cursor through the CRT controller ports `0x3D4` / `0x3D5`
- Handles control characters: backspace, tab (8-column stops), newline, form feed (clear screen) and carriage return
- `defilement()` scrolls the screen up by one line with `memmove`
- `console_putbytes()` connects the provided `printf` to the screen

### Timer & interrupts (`time.c`, `traitants.S`)
- Sets the **PIT** (Programmable Interval Timer) to 50 Hz by writing the quartz divisor to ports `0x43` / `0x40`
- **Registers an interrupt handler** by writing a gate entry for vector 32 into the interrupt table (IDT) at `0x1000`
- Unmasks IRQ0 on the interrupt controller (PIC, port `0x21`) and acknowledges each interrupt (EOI on port `0x20`)
- The assembly entry point `traitant_IT_32` saves the scratch registers, calls the C handler `tic_PIT` and returns with `iret`
- `tic_PIT` keeps an `HH:MM:SS` clock and shows it in the top-right corner of the screen

### Processes & scheduling (`processus.c`, `ctx_sw.S`)
- A fixed process table (`struct processus`): PID, name, state (`elu` = running / `activable` = ready / `endormi` = sleeping), saved registers and a 512-word stack for each process
- `cree_processus()` sets up a new process's stack so that the first context switch "returns" into its entry function
- **Context switch** (`ctx_sw`) saves and restores `ebx`, `esp`, `ebp`, `esi` and `edi`
- **Preemptive round-robin scheduling**: the timer interrupt calls `ordonnance()` once per second, which passes the CPU to the next process in the table
- An `idle` process (`sti; hlt; cli`), plus `mon_pid()` / `mon_nom()` helpers
- A first version of `dors(n)` (sleep) records the wake-up time, but the scheduler does not yet skip sleeping processes

## Tech stack

C99 (freestanding: `-nostdinc`, no host C library) · x86 (i386) assembly (AT&T syntax) · GNU `ld` with a custom linker script (`kernel.lds`) · QEMU · GDB · Make

## Project structure

The repository is flat. The main files are:

```
start.c          kernel entry point (kernel_start) + VGA console driver
time.c / .h      PIT setup, IDT entry, IRQ masking, clock display
traitants.S      timer interrupt entry point (assembly)
processus.c / .h process table, creation, scheduler, sleep
ctx_sw.S         context switch (assembly)
crt0.S, kernel.lds, processor_structs.c, printf.c, string.c, ...
                 boot code, linker script and minimal libc provided by the course
Makefile
```

## Build & run

Requirements: Linux, `gcc` with 32-bit support (`-m32`), GNU `ld`, `objcopy` and `qemu-system-i386`.

```sh
make            # cleans, then builds kernel.bin
make run        # boots kernel.bin in QEMU
make debug      # starts QEMU paused with a GDB server on tcp::1234
```

To debug with GDB, run `make debug`, then in another terminal start `gdb kernel.bin` and enter `target remote :1234`.

The Makefile expects QEMU at `/usr/bin/qemu-system-i386`. Change the `QEMU` variable if yours is installed elsewhere.
