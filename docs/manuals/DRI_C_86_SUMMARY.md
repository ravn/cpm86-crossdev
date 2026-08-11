# Digital Research C (DR C) for CP/M-86 — working summary

Analysis of `DRI_C_Programming_86.txt` (extracted from
`DRI_C_Programming_86.pdf`, "C Language Programmer's Guide for the CP/M-86
Family of Operating Systems", Digital Research, 1983). This is the reference
for the DR C compiler that targets 8086/8088 (and by extension the 80186
RC759 under CP/M-86 / Concurrent CP/M-86). Page cites are the manual's own
`N-M` section/page numbers.

> Purpose: quick-reference for how DR C *does things* — compiler invocation,
> memory models, the C↔assembler ABI, and on-disk data layout — so future
> RC759 cross-dev work doesn't have to re-read 189 pages. Where the extracted
> text has scan-OCR noise, the meaning below is reconstructed and marked.

---

## 1. Toolchain / compiler invocation (§2.1)

Compiler driver is **`DRC`** (supervisory module `DRC.CMD`). It runs three
overlays + a merge util + linker:

| File          | Role                                        |
|---------------|---------------------------------------------|
| `DRC.CMD`     | supervisory driver                          |
| `DRC860.CMD`  | preprocessor (`-0` sets its drive)          |
| `DRC861.CMD`  | parser + code generator (`-1` sets drive)   |
| `DRC862.CMD`  | listing / disassembly merge (`-2` drive)    |
| `LINK86.CMD`  | linker (`-3` sets drive)                    |

Command form: `DRC sourcefile option-switches` (max 128 chars). `.C` filetype
is assumed. Stop mid-compile with any key → `Stop DRC (Y/N)?`.

**Option switches** (dash + reserved char, no space before param; switches are
case-insensitive under CP/M-86; full table §2.1.3 / App. B):

- `-a[files]` — auto-invoke LINK-86 at end; list extra obj/libs after `-a`
  (`[I]` = LINK-86 input file). `DRC PROGRAM -A` compiles + links with the
  small-model system library.
- `-b` — **big memory model** (default is small).
- `-d[name]` — like `#define name 1`, but defines in lowercase only.
- `-f` — use **8087** math coprocessor (else software float).
- `-h` — suppress sign-on banner.
- `-i[drive:]` — search drive for `#include` files.
- `-j` — disable short/long jump optimizer.
- `-l[name]` — program listing (default CON:).
- `-n` — disable code optimizer (faster compile).
- `-o[filename]` — object filename (`.OBJ` appended if no `.`).
- `-p` — run preprocessor only → `CTEMP.TOK`.
- `-q[number]` — code-generator node budget (default 500, min 100).
- `-r[name]` — reverse-assembly interlisting (default CON:).
- `-v[1..5]` — compiler message verbosity (1 general … 5 file+line per line).
- `-w[0..2]` — error display: w0 all, w1 suppress warnings, w2 suppress all.
- `-x` — call a lib routine to save/restore regs instead of inline entry/exit
  code → **smaller but slower; small model only**.
- `-z[drive:]` — temp work-file drive.
- `-0/-1/-2/-3[drive:]` — locate the overlay/linker modules (see table above).

`-p` output token file is `CTEMP.TOK`; there is also a "reverse preprocessor"
(§2.3) and `-r` interlisting (reverse assembly) for inspecting codegen.

### Cross-dev wrapper (this repo)
The `cpm86-crossdev` repo wraps DR's DOS/CP/M tools to run on macOS/Linux via
the **emu2** DOS emulator (and **tnylpo** for CP/M-80 tools like `asm86.com` /
`gencmd.com`). It also ships **Aztec C 3.4 / 4.2** and DR `rasm86/link86/lib86`.
So "DR C" here is one of several C paths; Watcom (open-watcom-v2) is the
modern 80186 path being added. See repo `README.md` + `bin/` wrappers.

---

## 2. Memory models (§2.4) — 8086 segmented

Three areas, each with a segment base register: **CS** (code), **DS** (data),
**SS** (stack); **ES** usually points at the heap.

- **Small model (default):** one `CGROUP` (code) + one `DGROUP` (data), each
  **≤ 64K**. Compiler emits group names `CGROUP`/`DGROUP` automatically; asm
  modules use the RASM-86 `GROUP` directive to place segments. Layout (low→high
  in DGROUP): data segments → heap (grows up) → stack (grows down from top).
  `ES = SS = DS`. **Pointers are 16-bit (offset only).** Calls/returns are
  **NEAR**.
- **Big model (`-b`):** data ≤ 64K (DGROUP), stack ≤ 64K (separate segment),
  but **code is many separate ≤64K segments** (total code limited only by
  memory); heap in ES limited only by memory. Do **not** put asm code segments
  in CGROUP. **Pointers are 32-bit** (offset in low word, segment in high
  word). Calls/returns are **FAR**. Stack size set in start-up routine,
  adjustable at link time.

For RC759 (80186): pick model by whether code exceeds 64K. Most CP/M-86
programs fit small model.

---

## 3. C ↔ assembler ABI (§5) — **the important part for codegen**

Assembler is **RASM-86** (ASM-86 subset), linked with LINK-86.

### 3.1 Naming (§5.1)
- External names significant to **8 chars** in DR C; portability advice: keep
  to 7. **DR C does *not* prefix `_`** on external names (unlike UNIX C).
- Some library routines have 1–2 leading `_`; don't call them directly except
  `_exit`.
- RASM-86 **upcases everything** unless `$nc` switch → therefore C code must
  declare asm functions/vars in **UPPERCASE**. `ASM_ROUTINE()` in C is seen by
  the compiler as `ASM_ROUT` (8-char trunc).

### 3.2 Calling asm from C (§5.2) / C from asm (§5.3)
- C→asm: `extern int FUNC1();` in C; `PUBLIC FUNC1` in asm.
- asm→C: `EXTRN FUNC1:NEAR` (small model) or `:FAR` (big model) in asm; C funcs
  need no explicit PUBLIC.

### 3.3 Argument passing (§5.4) — **caller-pushed, right-to-left, stack**
- All args on the **hardware stack**, pushed **right→left**, all **by value**.
- 1-byte args pass as a **word** (value in low byte).
- Multiword values (long, float, double): **high-order word pushed first**
  (so low word ends up at lower address / nearer top of stack).
- Small-model pointer = 1 word (offset). Big-model pointer = dword
  (offset low, segment high); **high (segment) word pushed first**.
- Called routine must **preserve SI and DI** if it uses them.

**Standard entry/exit protocol** the compiler emits (frame pointer = BP):
```
FUNCTION:               ; entry
        PUSH  BP        ; save caller frame ptr
        MOV   BP,SP     ; new frame ptr
        PUSH  DI        ; save register vars
        PUSH  SI
        SUB   SP,nnn    ; allocate locals
        ...             ; body
        LEA   SP,-4[BP] ; drop locals, point at saved SI/DI  (OCR: "4[BP]")
        POP   SI        ; restore register vars
        POP   DI
        POP   BP        ; restore frame ptr
        RET             ; RETF for big model
```
Args are accessed at positive offsets from BP above the saved BP/return addr;
locals at negative offsets. (Manual shows worked stack frames Fig 5-1 small /
5-2 big for `TESTFUNC(int,long,char,float,double,char*)`.)

### 3.4 Return values (§5.5) — **Table 5-1**
| C type                                   | Register(s)                                   |
|------------------------------------------|-----------------------------------------------|
| `int`, `char`, `short`, small-model ptr  | **AX**                                        |
| `long`, `float`, big-model ptr           | **high word / segment in BX, low word / offset in AX** |
| `double`                                 | **DX** (high), **CX** (high-mid), **BX** (low-mid), **AX** (low) |

### 3.5 External data (§5.6)
- Each external C variable becomes a **separate COMMON data segment**; the
  variable name is the *segment* name. In asm you must declare each shared var
  as its own segment with COMMON combine type and allocate space; you can't
  reference the segment name directly, so re-name inside the routine.

### 3.6 Direct OS calls from C (§3.2)
- `ret = BDOS(func, arg)` — `func` = BDOS function number (int), `arg` depends
  on the call. **Returns a `char`.** For non-char returns you must write your
  own asm interface routine. (This is the hook for RC759 XIOS/BDOS-level work.)

---

## 4. Internal data representation (§6) — **little-endian, IEEE float**

- **char**: 8-bit **unsigned**, 0–255 (always positive). (§6.1)
- **short/int**: 16-bit two's complement, −32768..32767; `unsigned int` 0..65535.
  Stored **low byte at low memory** (little-endian). (§6.2)
- **long**: 32-bit two's complement, signed only (**no `unsigned long`**),
  −2,147,483,648..2,147,483,647; little-endian byte0..byte3. (§6.2)
- **float** (single): 4 bytes **IEEE-754 binary32** — 1 sign, 8-bit exponent
  (bias 0x7F), 23-bit mantissa + implicit leading 1. ~7 decimal digits. (§6.3)
- **double**: 8 bytes **IEEE-754 binary64** — 1 sign, 11-bit exponent
  (bias 0x3FF), 52-bit mantissa + implicit 1. ~15 digits. (§6.4)
- **All float arithmetic is done in double precision**; singles are padded to
  double for computation and rounded back on assignment. (§6.4)
- **Pointer**: small model = 1 word offset; big model = dword (offset, segment).
  (§6.5)

---

## 5. I/O model (§4)

Everything is a file (devices too). Two access styles + 3 standard files.

- **Regular (low-level) access (§4.1):** direct OS entry, no buffering. Create
  with `creat`/`creata`(ASCII)/`creatb`(binary); open with
  `open`/`opena`/`openb`. Returns a **file descriptor** 0–15 (CP/M-86).
  ASCII files: CR/LF line ends + CTRL-Z (0x1A) EOF, auto-translated to/from
  UNIX-style bare LF on read/write; binary files have no auto EOF — program
  tracks it. Funcs: `close creat creata creatb lseek open opena openb read
  tell unlink write`.
- **Stream (buffered) access (§4.2):** 512-byte buffer; stream = pointer to a
  control struct (FILE). Funcs: `fopen/fopena/fopenb freopen* fclose fflush
  fread fwrite fgetc/getc/getchar/getw/getl fputc/putc/putchar/putw/putl
  fgets/gets fputs/puts fprintf/printf fscanf/scanf fseek/ftell/rewind feof
  ferror fileno fdopen ungetc`.
- **Peripheral device names (§4.3):** `CON:` console, `LST:` list device.
- **Standard I/O (§4.4):** `stdin`(fd 0), `stdout`(fd 1), `stderr`(fd 2), all
  open at start, connected to terminal. `STDIO.H` (which pulls in `PORTAB.H`)
  **must be `#include`d by any program using a library function.** Redirect on
  the command line with `<` / `>` (e.g. `prog <indat >lst:`).

Storage-class macros (§3): `REG` register, `LOCAL` auto, `MLOCAL` module
static, `GLOBAL` global def, `EXTERN` global ref. **Library function names must
be typed lowercase.**

---

## 6. Start-up & stand-alone (§2.2)

- Start-up routine (`START` in the system lib, source `STARTUP.A86`) inits
  stack ptr, segment regs, heap, then calls `main()`; on return flushes
  buffers, closes files, frees storage, returns to OS.
- **Stand-alone / systems programs:** link with a *custom* start-up (replacing
  `START`) that sets up your target environment; your start-up module must be
  linked **first**. Machine-support subroutines (long divide/shift) are pulled
  in implicitly by the compiler; you can't call them explicitly. This is the
  path for a bare-metal/OS-level 80186 target.

---

## 7. Overlays (§7) — brief
Programs can use overlays; `LINK-86` command lines build them (§7.2), with
general constraints in §7.3. Relevant only if RC759 programs outgrow memory;
revisit the chapter directly if needed.

---

## 8. Appendices (pointers, not expanded here)
- **A** System Library Routine Summary — one-line prototype per function.
- **B** Compiler Option Summary — same table as §2.1.3.
- **C** Error Messages.
- **D** Variations among Compilers — read before assuming portability of any
  DR-C-specific behavior above.
- **E** C Style Guide (module size, header files, naming, module layout).
- **F** Sample C Modules.

---

## 9. RC759-relevant takeaways
1. **ABI to match for any hand asm / codegen:** args caller-pushed R→L on
   stack, multiword high-word-first; returns in AX (16-bit) / BX:AX (32-bit &
   far ptr) / DX:CX:BX:AX (double); preserve SI,DI; BP frame; NEAR (small) vs
   FAR (big).
2. **No leading `_` on externs; uppercase-only asm symbols; 8-char
   significance** — different from the z80/z88dk `_`-prefix world.
3. **Little-endian, IEEE-754 binary32/binary64, char is unsigned, no
   `unsigned long`.**
4. **OS access via `BDOS(func,arg)->char`**; richer returns need a custom asm
   thunk — the natural seam for RC759 XIOS (Int 28h) work already documented in
   `open-watcom-v2/contrib/ravn/EMULATION-PLAN.md`.
5. Small model (16-bit ptrs, ≤64K+≤64K) is the default and the common CP/M-86
   case; only go big model if code > 64K.

## OCR caveats
The extracted text is a clean text layer but has scanning artifacts: negative
signs/exponents split from digits (e.g. `1.18*10**38` means `1.18e-38`), stray
`_`/spaces in identifiers, and math ranges garbled. Trust the *structure* here;
re-open the PDF for exact figures/tables when precision matters.
