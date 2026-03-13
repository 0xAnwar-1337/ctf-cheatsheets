# SageMath Cheatsheet

### Practical reference for CTF, Cryptography & Reverse Engineering

A concise **practical SageMath cheatsheet** focused on:

-   CTF cryptography challenges
-   Number theory
-   RSA attacks
-   Lattices & LLL
-   Finite fields
-   ECC basics

Works in: - `sage` REPL - `.sage` scripts - `sage -python` - Jupyter
with Sage kernel

------------------------------------------------------------------------

# Table of Contents

-   Installation
-   Quick Start
-   Integer & Modular Arithmetic
-   Modular Rings (Zmod)
-   Primality & Factoring
-   Polynomials & Finite Fields
-   Linear Algebra & LLL
-   Continued Fractions
-   RSA Helpers
-   Useful Conversions
-   Random Helpers
-   ECC Basics
-   CAS / Symbolic Math
-   Practical Example (Wiener Attack)
-   Tips & Gotchas

------------------------------------------------------------------------

# Installation

### Conda (recommended)

``` bash
conda create -n sage -c conda-forge sage
conda activate sage
```

### Docker

``` bash
docker run -it --rm sagemath/sagemath:latest
```

### Online

-   CoCalc
-   SageMathCell

------------------------------------------------------------------------

# Quick Start

``` python
from sage.all import *
```

Important rings:

    ZZ  -> integers
    QQ  -> rationals
    RR  -> real numbers
    GF(p) -> finite field

Example:

``` python
a = Integer(123)
b = ZZ(456)

a + b
```

Convert Sage → Python:

``` python
int(a)
```

------------------------------------------------------------------------

# Integer & Modular Arithmetic

``` python
g = gcd(a, b)

x, y, d = xgcd(a, b)

inverse_mod(a, m)

pow(a, b, m)

crt([a1, a2], [m1, m2])

solve_mod(x^2 == 4, 7)
```

------------------------------------------------------------------------

# Modular Rings (Zmod)

``` python
R = Zmod(17)

a = R(5)
b = R(3)

a + b
a * b
a^5

a.inverse()
```

------------------------------------------------------------------------

# Primality & Factoring

``` python
is_prime(n)

next_prime(n)

previous_prime(n)

factor(n)

factorint(n)
```

Example:

``` python
factor(360)
```

------------------------------------------------------------------------

# Polynomial Rings

``` python
R.<x> = PolynomialRing(ZZ)

f = x^4 + 2*x^3 - x + 7

f.factor()
```

------------------------------------------------------------------------

# Finite Fields

``` python
F = GF(101)

a = F(10)

a^5
```

Extension field:

``` python
K = GF(2^8, 'a')

alpha = K.gen()
```

------------------------------------------------------------------------

# Square Roots Modulo

``` python
sqrt_mod(a, p)

sqrt_mod(a, p, all=True)
```

------------------------------------------------------------------------

# Discrete Logarithm

``` python
g = Mod(5, 97)

h = g^13

d = discrete_log(h, g)
```

------------------------------------------------------------------------

# Linear Algebra

``` python
M = Matrix(ZZ, [[105,821],[102,816]])

M.det()

M.rank()

M.right_kernel().basis()
```

------------------------------------------------------------------------

# Lattice Reduction (LLL)

``` python
B = Matrix(ZZ, [
[105,821,11],
[102,816,9],
[1,1,1]
])

B.LLL()
```

Used in: - Coppersmith - HNP - partial key leaks

------------------------------------------------------------------------

# Continued Fractions

``` python
cf = continued_fraction(426/127)

convergents(cf)
```

------------------------------------------------------------------------

# RSA Helper Operations

``` python
inverse_mod(e, phi)

pow(m, e, n)

crt([...], [...])
```

------------------------------------------------------------------------

# Random Helpers

``` python
randint(1,100)

random_prime(2^512)
```

------------------------------------------------------------------------

# Useful Conversions

### bytes -\> int

``` python
def bytes_to_long(b):
    return int.from_bytes(b, 'big')
```

### int -\> bytes

``` python
def long_to_bytes(n):
    length = (n.bit_length() + 7)//8
    return n.to_bytes(length,'big')
```

------------------------------------------------------------------------

# ECC Basics

``` python
E = EllipticCurve(GF(p), [a, b])
```

Points:

``` python
P = E.random_point()

Q = 2*P
```

Discrete log (small curves only):

``` python
E.discrete_log(Q, P)
```

------------------------------------------------------------------------

# CAS / Symbolic Math

``` python
x = var('x')

expr = x^3 - 2*x + 1

diff(expr, x)

solve(expr == 0, x)
```

------------------------------------------------------------------------

# Practical Example --- Wiener Attack

``` python
from sage.all import *

def wiener_attack(e, n):

    cf = continued_fraction(e/n)

    for conv in convergents(cf):

        k = conv.numerator()
        d = conv.denominator()

        if k == 0:
            continue

        if (e*d - 1) % k != 0:
            continue

        phi = (e*d - 1)//k

        s = n - phi + 1
        disc = s*s - 4*n

        if disc >= 0 and is_square(disc):

            t = isqrt(disc)

            p = (s + t)//2
            q = (s - t)//2

            if p*q == n:
                return d, p, q

    return None
```

------------------------------------------------------------------------

# Tips & Gotchas

✔ Convert Sage integers with `int(x)`

✔ For heavy factoring use:

``` python
pari(...)
```

✔ Run Python scripts with Sage:

``` bash
sage -python script.py
```

✔ LLL and Coppersmith can consume large memory.

✔ When mixing libraries (gmpy2, pycryptodome), run inside
`sage -python`.
