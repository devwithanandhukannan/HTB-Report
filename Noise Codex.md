# Noise Codex — Complete Mathematical Breakdown & Solution Guide

A step-by-step teaching guide explaining the mathematics and cryptanalysis behind the **Affine Noise Cipher** challenge from Hack The Box.

---

## 1. Challenge Overview

The challenge encrypts a secret string (`flag`) bit-by-bit.

1. The message text is converted into a binary bit string:
   $$\text{"A"} \longrightarrow 65 \longrightarrow 01000001_2$$
2. For each individual bit $b \in \{0, 1\}$, a large ciphertext number $c$ is computed:

$$c = p \cdot q + 2r + b$$

### Variables & Their Roles:
| Variable | Name in Code | Mathematical Role | Approximate Size |
| :--- | :--- | :--- | :--- |
| **$p$** | `anchor` | Secret prime (shared for all bits) | $1024\text{ bits} \approx 300\text{ digits}$ |
| **$q$** | `distortion` | Random multiplier (unique per bit) | $2048\text{ bits} \approx 600\text{ digits}$ |
| **$r$** | `entropy` | Random noise (unique per bit) | $256 - 512\text{ bits} \approx 150\text{ digits}$ |
| **$b$** | `bit` | The secret flag bit ($0$ or $1$) | $1\text{ bit}$ |
| **$c$** | `ciphertext` | The final output saved to file | $3072\text{ bits} \approx 900\text{ digits}$ |

---

## 2. The Core Mathematical Weaknesses

There are **two critical mathematical properties** that allow us to break this encryption completely.

---

### Flaw 1: Noise Cancellation Modulo 2

Look at the term $(2r + b)$ added to $p \cdot q$:
- $p \approx 2^{1024}$ is huge.
- $2r + b \le 2 \cdot 2^{512} + 1 < 2^{513} + 1$.

Because $2r + b < p$, taking the remainder of $c$ divided by $p$ gives:

$$c \bmod p = (p \cdot q + 2r + b) \bmod p = 2r + b$$

Now observe the number $2r$:
- $2 \times r$ is **always even** (a multiple of 2).
- Therefore, $(2r \bmod 2) = 0$.

If we take $(c \bmod p) \bmod 2$:

$$(c \bmod p) \bmod 2 = (2r + b) \bmod 2 = \underbrace{(2r \bmod 2)}_{0} + (b \bmod 2) = b$$

> 💡 **Key Takeaway**: If we can discover the secret prime $p$, we can extract every secret bit $b$ using just one simple line:
> ```python
> b = (c % p) % 2
> ```

---

### Flaw 2: Approximate Common Divisor (ACD)

How do we find $p$ when we only have the ciphertexts $c_0, c_1, c_2, \dots$?

Every ciphertext is an **approximate multiple** of $p$:
$$c_0 = p \cdot q_0 + e_0$$
$$c_1 = p \cdot q_1 + e_1$$
$$c_2 = p \cdot q_2 + e_2$$

*(where $e_i = 2r_i + b_i$ is the small noise)*

If we divide $c_1$ by $c_0$:

$$\frac{c_1}{c_0} = \frac{p \cdot q_1 + e_1}{p \cdot q_0 + e_0} \approx \frac{q_1}{q_0}$$

Because the errors $e_0, e_1$ are tiny compared to $c_0, c_1$, the ratio of the ciphertexts $\frac{c_1}{c_0}$ is extremely close to the rational number $\frac{q_1}{q_0}$.

This is the classic **Simultaneous Diophantine Approximation** problem, which can be solved using **Lattice Reduction (LLL)**.

---

## 3. The Lattice Reduction (LLL) Mathematics

### Building the Lattice Matrix

Let's pick $k$ ciphertexts $c_1, c_2, \dots, c_k$ and pair them with a base ciphertext $c_0$.

We want to find integers $q_0, q_1, \dots, q_k$ such that:
$$q_0 c_i - q_i c_0 = q_0 (p q_i + e_i) - q_i (p q_0 + e_0) = q_0 e_i - q_i e_0$$

Notice how small the right side is:
- $c_0 \approx 2^{3072}$ (massive)
- $q_0 e_i - q_i e_0 \approx 2^{2048 + 513} = 2^{2561}$ (much smaller than $c_0$!)

We construct the following matrix $M$ (where $E \approx 520$ is a scaling factor to balance vector lengths):

$$M = \begin{pmatrix}
2^E & c_1 & c_2 & \dots & c_k \\
0 & -c_0 & 0 & \dots & 0 \\
0 & 0 & -c_0 & \dots & 0 \\
\vdots & \vdots & \vdots & \ddots & \vdots \\
0 & 0 & 0 & \dots & -c_0
\end{pmatrix}$$

### What Happens During Vector Multiplication?

Consider the integer vector $\mathbf{u} = (q_0, q_1, q_2, \dots, q_k)$. When multiplied by matrix $M$:

$$\mathbf{u} \cdot M = \begin{pmatrix}
q_0 \cdot 2^E, & (q_0 c_1 - q_1 c_0), & (q_0 c_2 - q_2 c_0), & \dots, & (q_0 c_k - q_k c_0)
\end{pmatrix}$$

Substitute $q_0 c_i - q_i c_0 = q_0 e_i - q_i e_0$:

$$\mathbf{v} = \begin{pmatrix}
q_0 \cdot 2^E, & (q_0 e_1 - q_1 e_0), & (q_0 e_2 - q_2 e_0), & \dots, & (q_0 e_k - q_k e_0)
\end{pmatrix}$$

Every component of this vector $\mathbf{v}$ has magnitude around $2^{2561}$, whereas generic vectors in this lattice have sizes around $2^{3072}$.

### Finding the Short Vector with LLL
The **LLL (Lenstra–Lenstra–Lovász)** algorithm efficiently finds unusually short vectors in a lattice. 

When LLL runs on matrix $M$, it immediately finds vector $\mathbf{v}$. From the very first entry of $\mathbf{v}$:

$$\text{First Entry} = q_0 \cdot 2^E \implies q_0 = \frac{\text{First Entry}}{2^E}$$

Once we know $q_0$, finding $p$ is simple integer division:

$$p = \left\lfloor \frac{c_0}{q_0} \right\rfloor$$

---

## 4. Step-by-Step Attack Flow

```text
 ┌─────────────────────────┐
 │   Read output.txt       │  --> Load 328 ciphertext numbers
 └───────────┬─────────────┘
             │
             ▼
 ┌─────────────────────────┐
 │  Build Lattice Matrix   │  --> Use c0 and 5 other ciphertexts (dimension 6)
 └───────────┬─────────────┘
             │
             ▼
 ┌─────────────────────────┐
 │       Run LLL           │  --> Reduces lattice and isolates the short vector
 └───────────┬─────────────┘
             │
             ▼
 ┌─────────────────────────┐
 │   Extract q0 & p        │  --> q0 = vector[0] >> E,  p = c0 // q0
 └───────────┬─────────────┘
             │
             ▼
 ┌─────────────────────────┐
 │   Recover All Bits      │  --> For every c in output: bit = (c % p) % 2
 └───────────┬─────────────┘
             │
             ▼
 ┌─────────────────────────┐
 │   Binary -> ASCII Flag  │  --> 8 bits = 1 character -> "HTB{...}"
 └─────────────────────────┘
```

---

## 5. Complete Python Solver (`solve.py`)

Below is the complete implementation in Python using `mpmath` for fast high-precision lattice reduction:

```python
import ast
import mpmath
import time

# Set precision for float operations in Gram-Schmidt orthogonalization
mpmath.mp.dps = 200

def lll(basis, delta=0.75):
    """
    Standard Lenstra-Lenstra-Lovasz (LLL) lattice basis reduction.
    Reduces the basis vectors to find the shortest non-zero vectors.
    """
    n = len(basis)
    m = len(basis[0])
    b = [list(row) for row in basis]
    delta = mpmath.mpf(delta)
    
    b_star = [[mpmath.mpf(0)] * m for _ in range(n)]
    mu = [[mpmath.mpf(0)] * n for _ in range(n)]
    B = [mpmath.mpf(0)] * n
    
    def update_row(i):
        v = [mpmath.mpf(x) for x in b[i]]
        for j in range(i):
            denom = B[j]
            if denom != 0:
                dot = sum(mpmath.mpf(b[i][l]) * b_star[j][l] for l in range(m))
                mu[i][j] = dot / denom
            else:
                mu[i][j] = mpmath.mpf(0)
            for l in range(m):
                v[l] -= mu[i][j] * b_star[j][l]
        b_star[i] = v
        mu[i][i] = mpmath.mpf(1)
        B[i] = sum(v[l] ** 2 for l in range(m))
        for j in range(i + 1, n):
            denom = B[i]
            if denom != 0:
                mu[j][i] = sum(mpmath.mpf(b[j][l]) * b_star[i][l] for l in range(m)) / denom

    for i in range(n):
        update_row(i)

    k = 1
    while k < n:
        # Size reduction step
        for j in range(k - 1, -1, -1):
            if abs(mu[k][j]) > 0.5:
                q = int(round(float(mu[k][j])))
                if q != 0:
                    for l in range(m):
                        b[k][l] -= q * b[j][l]
                    update_row(k)
                    
        # Lovasz condition check
        if B[k] >= (delta - mu[k][k - 1] ** 2) * B[k - 1]:
            k += 1
        else:
            # Swap vectors
            b[k], b[k - 1] = b[k - 1], b[k]
            update_row(k - 1)
            update_row(k)
            k = max(k - 1, 1)
            
    return b

def solve():
    print("[*] Reading ciphertext data from output.txt...")
    with open('output.txt', 'r') as f:
        ciphertexts = ast.literal_eval(f.read())

    # Step 1: Select a subset of samples (dimension 6 is sufficient)
    samples = ciphertexts[:6]
    c0 = samples[0]
    others = samples[1:]

    # Step 2: Construct the Diophantine approximation matrix
    E = 520
    k_dim = len(others)
    basis = []
    row0 = [1 << E] + others
    basis.append(row0)
    for i in range(k_dim):
        row = [0] * (k_dim + 1)
        row[i + 1] = -c0
        basis.append(row)

    # Step 3: Run LLL to find the short vector
    print(f"[*] Running LLL on lattice of dimension {len(basis)}...")
    t0 = time.time()
    reduced = lll(basis)
    print(f"[*] LLL completed in {time.time() - t0:.2f}s")

    # Step 4: Extract q0, compute p, and decrypt the flag
    for row in reduced:
        val = abs(row[0])
        q0 = val >> E
        if q0 > 0:
            p = c0 // q0
            if p.bit_length() == 1024:
                print(f"[+] Successfully recovered 1024-bit anchor prime p:\n    p = {p}\n")
                
                # Step 5: Decrypt all bits
                bits = []
                for c in ciphertexts:
                    bit = (c % p) % 2
                    bits.append(str(bit))
                bitstr = ''.join(bits)
                
                # Convert 8-bit binary chunks into ASCII characters
                flag = ''.join(chr(int(bitstr[i:i+8], 2)) for i in range(0, len(bitstr), 8))
                print(f"[+] FLAG: {flag}")
                return flag

if __name__ == '__main__':
    solve()
```

---

## 6. Running the Solver & Verifying the Flag

Execute the script in your terminal:

```bash
python3 solve.py
```

### Execution Output:

```text
[*] Reading ciphertext data from output.txt...
[*] Running LLL on lattice of dimension 6...
[*] LLL completed in 6.10s
[+] Successfully recovered 1024-bit anchor prime p:
    p = 164402719672345215073829428735717557756567193421978009297551706902713108670016609570405874271706278725280615925611348168195172295728363418512008345720021892377239594133562822511728463730646210357687537560736691914954360902850522686962670978389935704755300357272786671321093585282806313799135559664510432674677

[+] FLAG: HTB{LLL_r3c0V3R_7H3_tRu7H_fR0m_7H3_n0153}
```
