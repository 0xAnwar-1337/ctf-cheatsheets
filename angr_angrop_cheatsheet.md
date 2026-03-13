# angr & angrop Cheatsheet
### Practical reference for CTF, Reverse Engineering & Binary Exploitation

A concise **practical cheatsheet** for:

- **angr**: binary analysis + symbolic execution
- **angrop**: automatic ROP chain generation on top of angr

Useful for:

- CTF reversing
- crackmes
- symbolic execution
- finding inputs to reach success states
- basic ROP automation

---

# Table of Contents

- Installation
- Quick Start
- Loading a Binary
- Symbolic Input
- States
- Simulation Manager
- Finding Paths
- stdin / argv / file input
- Hooks
- Memory / Registers
- Constraints
- Solvers
- Common CTF Patterns
- angrop Basics
- Building ROP Chains
- Practical Tips
- Minimal Examples

---

# Installation

## angr

```bash
pip install angr
```

## angrop

```bash
pip install angrop
```

> `angrop` works on top of `angr`, so you usually install both.

---

# Quick Start

```python
import angr
import claripy
```

- `angr` loads and analyzes binaries
- `claripy` creates symbolic values and constraints

---

# Loading a Binary

```python
import angr

# auto_load_libs=False is common in CTFs
proj = angr.Project("./chall", auto_load_libs=False)
```

Useful options:

```python
proj = angr.Project(
    "./chall",
    auto_load_libs=False,   # do not load shared libraries
    main_opts={"base_addr": 0x400000}  # useful for blobs / special cases
)
```

---

# Creating States

A **state** represents one possible execution snapshot.

## Entry state

```python
state = proj.factory.entry_state()
```

Starts from the real program entry point.

## Full init state

```python
state = proj.factory.full_init_state()
```

Good when you want runtime initialization to happen.

## Blank state

```python
state = proj.factory.blank_state(addr=0x401000)
```

Useful when starting from a specific address.

## Call state

```python
state = proj.factory.call_state(0x4011A0, 0x1234, 0x5678)
```

Useful for directly simulating a function call.

---

# Symbolic Values

Use `claripy` to create symbolic variables.

## Symbolic bitvector

```python
sym_x = claripy.BVS("sym_x", 32)   # 32-bit symbolic value
```

## Symbolic bytes / input buffer

```python
flag = claripy.BVS("flag", 8 * 32)   # 32 bytes
```

## Concrete value

```python
value = claripy.BVV(0x41414141, 32)
```

---

# Symbolic stdin

```python
flag = claripy.BVS("flag", 8 * 32)

state = proj.factory.full_init_state(stdin=flag)
```

Recover the result:

```python
model = state.solver.eval(flag, cast_to=bytes)
print(model)
```

---

# Adding Constraints

```python
# force printable ASCII
for byte in flag.chop(8):
    state.solver.add(byte >= 0x20)
    state.solver.add(byte <= 0x7e)
```

Specific prefix:

```python
state.solver.add(flag.get_byte(0) == ord("F"))
state.solver.add(flag.get_byte(1) == ord("L"))
state.solver.add(flag.get_byte(2) == ord("A"))
state.solver.add(flag.get_byte(3) == ord("G"))
```

---

# Simulation Manager

The **SimulationManager** explores paths.

```python
simgr = proj.factory.simgr(state)
```

Run everything:

```python
simgr.run()
```

Step once:

```python
simgr.step()
```

Explore until target:

```python
simgr.explore(find=0x401234)
```

Find and avoid:

```python
simgr.explore(find=0x401234, avoid=0x401210)
```

---

# Finding a Winning Path

```python
import angr
import claripy

proj = angr.Project("./chall", auto_load_libs=False)

flag = claripy.BVS("flag", 8 * 32)
state = proj.factory.full_init_state(stdin=flag)

for b in flag.chop(8):
    state.solver.add(b >= 0x20)
    state.solver.add(b <= 0x7e)

simgr = proj.factory.simgr(state)
simgr.explore(find=lambda s: b"Correct" in s.posix.dumps(1),
              avoid=lambda s: b"Wrong" in s.posix.dumps(1))

if simgr.found:
    found = simgr.found[0]
    print(found.solver.eval(flag, cast_to=bytes))
```

---

# Reading stdin / stdout / stderr

```python
# stdin bytes seen by the program
state.posix.dumps(0)

# stdout
state.posix.dumps(1)

# stderr
state.posix.dumps(2)
```

Very useful in CTFs.

---

# argv Input

```python
arg1 = claripy.BVS("arg1", 8 * 16)

state = proj.factory.full_init_state(args=["./chall", arg1])
```

Recover:

```python
print(state.solver.eval(arg1, cast_to=bytes))
```

---

# Symbolic File Input

```python
data = claripy.BVS("file_data", 8 * 64)

state = proj.factory.full_init_state(
    fs={"input.txt": angr.SimFile("input.txt", content=data)}
)
```

---

# Memory Access

```python
# read 4 bytes from memory
x = state.memory.load(0x404000, 4)

# write 4 bytes
state.memory.store(0x404000, claripy.BVV(0xdeadbeef, 32))
```

---

# Register Access

```python
# read register
rax = state.regs.rax
rip = state.regs.rip

# write register
state.regs.rdi = claripy.BVV(0x1337, 64)
```

---

# Solving Expressions

```python
x = state.regs.rax

# evaluate as integer
print(state.solver.eval(x))

# evaluate several solutions
print(state.solver.eval_upto(x, 5))

# check satisfiable
print(state.solver.satisfiable())
```

As bytes:

```python
print(state.solver.eval(flag, cast_to=bytes))
```

---

# Hooks

Hooking replaces a function or address with your own behavior.

## Hook an address

```python
@proj.hook(0x401000, length=5)
def skip_check(state):
    state.regs.rax = 1
```

## Hook a symbol

```python
class ReturnZero(angr.SimProcedure):
    def run(self):
        return 0

proj.hook_symbol("strcmp", ReturnZero())
```

Useful for:

- skipping sleep / anti-debug
- bypassing checks
- simplifying library behavior

---

# Common CTF Patterns

## Find address that prints success

```python
simgr.explore(find=lambda s: b"win" in s.posix.dumps(1).lower())
```

## Avoid failure text

```python
simgr.explore(
    find=lambda s: b"correct" in s.posix.dumps(1).lower(),
    avoid=lambda s: b"wrong" in s.posix.dumps(1).lower()
)
```

## Start from a function instead of the entry point

```python
state = proj.factory.call_state(0x4011A0, claripy.BVS("x", 64))
```

## Start from a known block

```python
state = proj.factory.blank_state(addr=0x401080)
```

---

# Useful Exploration Techniques

```python
simgr = proj.factory.simgr(state)

# move one basic block at a time
simgr.step()

# run until no active states remain
simgr.run()
```

You may also inspect stashes:

```python
print(simgr.active)
print(simgr.deadended)
print(simgr.found)
print(simgr.errored)
```

---

# angr Options

Some useful state options:

```python
import angr

state = proj.factory.entry_state()

state.options.add(angr.options.ZERO_FILL_UNCONSTRAINED_MEMORY)
state.options.add(angr.options.ZERO_FILL_UNCONSTRAINED_REGISTERS)
```

Useful when angr complains about unconstrained data.

---

# Fast Patterns for Crackmes

## Symbolic password buffer

```python
pwd = claripy.BVS("pwd", 8 * 20)
state = proj.factory.full_init_state(stdin=pwd)
```

## Null-terminate input

```python
state.solver.add(pwd.get_byte(19) == 0)
```

## Restrict charset

```python
for b in pwd.chop(8):
    state.solver.add(b >= ord('0'))
    state.solver.add(b <= ord('z'))
```

---

# angrop Basics

`angrop` automates **ROP gadget discovery** and **chain generation**.

```python
import angr
import angrop

proj = angr.Project("./chall", auto_load_libs=False)

rop = proj.analyses.ROP()
rop.find_gadgets()
```

---

# Listing Gadgets

```python
for gadget in rop.rop_gadgets[:10]:
    print(gadget)
```

This helps inspect what the binary offers.

---

# Setting Registers with angrop

```python
chain = rop.set_regs(rax=0x3b, rdi=0x404100)
print(chain)
```

This is useful for building syscall-oriented chains.

---

# Writing to Memory with angrop

```python
chain = rop.write_to_mem(0x404100, b"/bin/sh\x00")
print(chain)
```

---

# Function Call Chain

```python
chain = rop.func_call("system", [0x404100])
print(chain)
```

If `/bin/sh\x00` is already placed at `0x404100`, this may generate a `system("/bin/sh")`-style chain.

---

# Raw Chain Bytes

```python
payload = chain.payload_str()
print(payload)
```

Often you append this to padding in a pwntools exploit.

---

# Combined Example (angrop)

```python
import angr
import angrop

proj = angr.Project("./chall", auto_load_libs=False)

rop = proj.analyses.ROP()
rop.find_gadgets()

chain = rop.write_to_mem(0x404100, b"/bin/sh\x00")
chain += rop.func_call("system", [0x404100])

payload = chain.payload_str()
print(payload)
```

---

# Common angrop Workflow

1. Load binary with `angr.Project`
2. Create `ROP` analysis
3. Call `find_gadgets()`
4. Build chain with helpers like:
   - `set_regs`
   - `write_to_mem`
   - `func_call`
5. Export bytes with `payload_str()`

---

# Practical Tips

- In CTFs, use `auto_load_libs=False` unless you really need shared libraries.
- Start simple: solve one function, not the whole binary.
- Use `call_state()` when you already know the interesting function.
- Add character constraints early to reduce solver chaos.
- Check `stdout` using `state.posix.dumps(1)` while exploring.
- Hook annoying functions like `sleep`, `puts`, `strcmp`, or custom checks when needed.
- For ROP, verify the generated chain in GDB or pwndbg; automation helps, but validation still matters.

---

# Minimal Example — angr

```python
import angr
import claripy

proj = angr.Project("./chall", auto_load_libs=False)

inp = claripy.BVS("inp", 8 * 16)
state = proj.factory.full_init_state(stdin=inp)

for b in inp.chop(8):
    state.solver.add(b >= 0x20)
    state.solver.add(b <= 0x7e)

simgr = proj.factory.simgr(state)
simgr.explore(find=lambda s: b"Success" in s.posix.dumps(1),
              avoid=lambda s: b"Fail" in s.posix.dumps(1))

if simgr.found:
    s = simgr.found[0]
    print(s.solver.eval(inp, cast_to=bytes))
```

---

# Minimal Example — angrop

```python
import angr
import angrop

proj = angr.Project("./chall", auto_load_libs=False)

rop = proj.analyses.ROP()
rop.find_gadgets()

chain = rop.set_regs(rax=0x3b)
print(chain)
print(chain.payload_str())
```

---

# Reminder

Use angr when you want to:

- reason about paths
- solve checks automatically
- recover inputs

Use angrop when you want to:

- automate gadget use
- build ROP chains faster
- reduce manual chain construction
