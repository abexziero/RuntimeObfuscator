# RuntimeObfuscator

**RuntimeObfuscator** is a tool for obfuscating functions in **C/C++ programs on Windows**. In simple terms, it hides what a function does by changing its code while the program is running, moves that code to another place in memory, and uses Windows exception handling to jump to the moved code when the function is called. This makes it much harder for someone to inspect the running program with a debugger or to analyse the program statically (for example, by looking at the `.text` section in a disassembler). Combine it with a packer and static analysis becomes even tougher.

---

## What it can do

* **Obfuscate code at runtime** — function code is altered while the program is running, so the original bytes aren’t sitting plainly in the `.text` section.
* **Use exception handling for execution** — it relies on Windows’ Vectored Exception Handling (VEH) to redirect execution to the moved code.
* **Add junk instructions** — inserts meaningless instructions where the original function used to be so disassembly is confusing.
* **Move functions around in memory** — relocates code so it’s not at a predictable address.
* **Basic anti-debug tricks** — causes traps and invalid memory accesses to frustrate debuggers.
* **Obfuscate control flow** — breaks up the normal flow of a function using VEH redirections.

---

## Installation

This runs on **Windows** and needs the Capstone disassembler ([https://github.com/capstone-engine/capstone])

```sh
# Clone the repository
git clone https://github.com/C5Hackr/RuntimeObfuscator.git
cd RuntimeObfuscator

# Install Capstone (if not already installed)
# On Windows make sure capstone_x86.lib and capstone_x64.lib are available to link against
```

---

## How to use it

Mark the function you want to hide with `BEGIN_PROTECTED_REGION` and `END_PROTECTED_REGION`, then call `ObfuscateFunction()` with the function address:

```c
#include <Windows.h>
#include "RuntimeObfuscator.h"

void SecretFunction()
{
    BEGIN_PROTECTED_REGION();
    printf("This is a hidden function!\n");
    END_PROTECTED_REGION();
}

int main()
{
    ObfuscateFunction((uintptr_t)SecretFunction);
    SecretFunction();
    return 0;
}
```

That tells RuntimeObfuscator to protect `SecretFunction` at runtime.

---

## How it works

RuntimeObfuscator uses self-modifying code techniques and moves code around at runtime. Here’s the process in straightforward steps:

### 1️⃣ Mark the function

You wrap the code with:

```c
BEGIN_PROTECTED_REGION();
// your code
END_PROTECTED_REGION();
```

Those markers let the tool see where the function starts and ends.

### 2️⃣ Obfuscate and relocate

* The tool copies the function to a freshly allocated memory area (this is the relocated copy).
* It overwrites the original function bytes with junk instructions.
* Often it will put an instruction that causes an access violation (or another fault) in the original location so that when the function is called an exception is raised.

### 3️⃣ VEH catches the exception and redirects

When the CPU hits that intentionally-bad instruction, Windows raises an exception. A VEH handler installed by RuntimeObfuscator notices the exception, figures out which function was being called, and redirects execution to the relocated (and usable) copy of the function.

Example of the VEH handler signature:

```c
LONG WINAPI VEHObfuscationHandler(PEXCEPTION_POINTERS exceptions)
```

---

## What the code looks like to a disassembler

After obfuscation, the original function area might look deliberately broken:

```asm
Obfuscated Function (Before Execution):
-------------------------------------
0x00400000:  MOV [INVALID_MEMORY], 1    ; triggers ACCESS_VIOLATION
0x00400011:  ?? ?? ?? ?? ??             ; junk bytes
0x00400016:  90 90 90 90 90             ; NOPs (more junk)

Relocated Function (At Runtime):
--------------------------------
0x7FFF0000:  40 53           ; PUSH RBX
0x7FFF0002:  49 8B D1        ; MOV RDX, R9
0x7FFF0005:  E9 78 56 34 12  ; JMP 0x12345678
0x7FFF0010:  CC              ; INT3 (used as a breakpoint for returns)
```

So the original location contains rubbish and a crash trigger; the real work happens in the relocated copy.

---

## Simple execution flow

```
Call obfuscated function
        ↓
Original function address has an invalid instruction -> Access Violation
        ↓
VEH handler runs, identifies the function -> redirects to relocated copy
        ↓
Relocated function executes normally
```

---

## Anti-debug techniques used

* Forces **access violation** or **illegal instruction** exceptions to hand control to the VEH handler rather than letting a debugger step through the normal code path.

---

## Known weakness — and how to make it tougher

### The weakness

A determined reverser could inject a DLL into the process, find the relocated region, and copy the working code back into the `.text` section. They’d then be able to dump the process and analyse it.

This is possible, but not trivial — they'd need to:

* Identify which relocated memory corresponds to which original function.
* Fix up relative jumps, calls, and memory references so the code works correctly when moved back.
* Recreate the original execution flow.

### How to improve it

Add another layer: partially encrypt the relocated function and decrypt only small parts when those parts are needed. In other words, keep the relocated code encrypted except for the exact bytes in use. This means even if an attacker finds the relocated memory, the code there will still be scrambled and much harder to restore.

Practical measures:

* Encrypt parts of the relocated function and only decrypt on-demand.
* Apply additional x86\_64-specific obfuscation at runtime.

With that extra layer, reconstructing the original function reliably becomes much harder.

---

## Limitations

* Works only on **Windows (x86/x64)**.
* Some operations don’t work inside relocated code (for example, certain system interactions or assumptions about absolute addresses may fail).

---

## Credits

* Uses the **Capstone** disassembly engine to help with analysing and relocating functions.
* Inspired by techniques used in runtime packers and virtualisation-based obfuscation.

---