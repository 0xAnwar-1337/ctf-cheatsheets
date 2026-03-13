# Z3 Solver Cheatsheet (z3py)

A concise **practical cheatsheet** for using the **Z3 SMT solver in
Python**.

Useful for:

-   CTF challenges
-   Reverse Engineering
-   Constraint solving
-   Symbolic execution helpers

------------------------------------------------------------------------

# Table of Contents

-   Installation
-   Import
-   Z3 Data Types
-   Arrays (Memory Modeling)
-   Creating a Solver
-   Checking Solver Result
-   Reading Values from Model
-   Bitwise Operations
-   Signed vs Unsigned Comparisons
-   Extract Bytes
-   Concat (Combine Values)
-   Zero Extension / Sign Extension
-   Conditional Expressions
-   Distinct
-   Push / Pop
-   Enumerating Multiple Solutions
-   Unsat Core
-   Optimization
-   Memory Modeling Example
-   Simplifying Expressions
-   Common CTF Pattern (Flag Solving)
-   Example --- Recover 4 Bytes
-   Practical Tips

------------------------------------------------------------------------

# Installation

``` bash
pip install z3-solver
```

------------------------------------------------------------------------

# Import

``` python
from z3 import *
```

------------------------------------------------------------------------

# Z3 Data Types

Every variable in Z3 must have a **type (called Sort)**.

### Integer

``` python
x = Int('x')   # mathematical integer
```

### Real

``` python
r = Real('r')  # real number
```

### Boolean

``` python
b = Bool('b')  # true / false
```

### BitVector (important for RE / CTF)

``` python
byte = BitVec('byte', 8)   # 8-bit value
reg  = BitVec('reg', 32)   # 32-bit register
reg64 = BitVec('reg64', 64) # 64-bit register
```

⚠️ In reverse engineering **BitVec is usually preferred over Int**.

------------------------------------------------------------------------

# Arrays (Memory Modeling)

``` python
mem = Array('mem', BitVecSort(32), BitVecSort(8))
# models memory: address -> byte
```

------------------------------------------------------------------------

# Creating a Solver

``` python
s = Solver()

x = BitVec('x', 32)

s.add(x > 10)
s.add(x < 20)

if s.check() == sat:
    m = s.model()
    print(m[x])
    print(m[x].as_long())
```

------------------------------------------------------------------------

# Checking Solver Result

``` python
result = s.check()

if result == sat:
    print("Solution found")
elif result == unsat:
    print("Constraints impossible")
else:
    print("Unknown result")
```

------------------------------------------------------------------------

# Reading Values from Model

``` python
m = s.model()

print(m[x])           # symbolic value
print(m[x].as_long()) # python integer
print(m.evaluate(x+5))
```

------------------------------------------------------------------------

# Bitwise Operations

``` python
a = BitVec('a', 32)
b = BitVec('b', 32)

a & b      # AND
a | b      # OR
a ^ b      # XOR
~a         # NOT

a << 3     # shift left
a >> 2     # arithmetic shift right
LShR(a,2)  # logical shift right

RotateLeft(a,7)
RotateRight(a,5)
```

------------------------------------------------------------------------

# Signed vs Unsigned Comparisons

### Signed

``` python
a < b
a > b
a <= b
a >= b
```

### Unsigned

``` python
ULT(a,b)
UGT(a,b)
ULE(a,b)
UGE(a,b)
```

------------------------------------------------------------------------

# Extract Bytes

``` python
v = BitVec('v', 32)

b0 = Extract(7,0,v)
b1 = Extract(15,8,v)
b2 = Extract(23,16,v)
b3 = Extract(31,24,v)
```

Useful when reconstructing flags.

------------------------------------------------------------------------

# Concat (Combine Values)

``` python
v = Concat(
    BitVecVal(ord('f'),8),
    BitVecVal(ord('l'),8),
    BitVecVal(ord('a'),8),
    BitVecVal(ord('g'),8)
)
```

------------------------------------------------------------------------

# Zero Extension / Sign Extension

### Zero extend

``` python
a = BitVec('a',8)
b = ZeroExt(24,a)
```

### Sign extend

``` python
b = SignExt(24,a)
```

------------------------------------------------------------------------

# Conditional Expressions

``` python
x = BitVec('x',32)

expr = If(x > 10, x + 1, x - 1)
```

------------------------------------------------------------------------

# Distinct

``` python
a = Int('a')
b = Int('b')
c = Int('c')

s.add(Distinct(a,b,c))
```

------------------------------------------------------------------------

# Push / Pop

``` python
s = Solver()

x = Int('x')

s.add(x > 0)

s.push()
s.add(x < 5)
s.check()

s.pop()
s.check()
```

------------------------------------------------------------------------

# Enumerating Multiple Solutions

``` python
solutions = []

while s.check() == sat:
    m = s.model()
    value = m[x].as_long()
    solutions.append(value)

    s.add(x != value)
```

------------------------------------------------------------------------

# Unsat Core

``` python
s = Solver()

a = Bool('a')

s.assert_and_track(a,'c1')
s.assert_and_track(Not(a),'c2')

s.check()

print(s.unsat_core())
```

------------------------------------------------------------------------

# Optimization

``` python
opt = Optimize()

x = Int('x')

opt.add(x >= 0)
opt.add(x <= 100)

opt.maximize(x)

opt.check()

print(opt.model()[x])
```

------------------------------------------------------------------------

# Memory Modeling Example

``` python
mem = Array('mem', BitVecSort(32), BitVecSort(8))

mem2 = Store(mem, BitVecVal(0x1000,32), BitVecVal(0x41,8))

value = Select(mem2, BitVecVal(0x1000,32))
```

------------------------------------------------------------------------

# Simplifying Expressions

``` python
x = Int('x')

expr = (x + 0) * 1

print(simplify(expr))
```

------------------------------------------------------------------------

# Common CTF Pattern (Flag Solving)

``` python
flag = [BitVec(f'f{i}',8) for i in range(32)]

for c in flag:
    s.add(c >= 32)
    s.add(c <= 126)
```

Convert solution to string:

``` python
if s.check() == sat:
    m = s.model()
    result = "".join(chr(m[c].as_long()) for c in flag)
    print(result)
```

------------------------------------------------------------------------

# Example --- Recover 4 Bytes

``` python
from z3 import *

v = BitVec('v',32)

s = Solver()

s.add(Extract(7,0,v) == ord('f'))
s.add(Extract(15,8,v) == ord('l'))
s.add(Extract(23,16,v) == ord('a'))
s.add(Extract(31,24,v) == ord('g'))

if s.check() == sat:
    print(hex(s.model()[v].as_long()))
```

------------------------------------------------------------------------

# Practical Tips

✔ Prefer **BitVec over Int**\
✔ Model inputs as **8-bit values**\
✔ Use **Extract** for bytes\
✔ Use **Concat** to rebuild values\
✔ Reduce constraints if solver is slow\
✔ Avoid nonlinear math when possible
