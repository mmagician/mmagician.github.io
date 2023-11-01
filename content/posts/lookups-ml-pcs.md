---
title: "Need for Speed: efficient multilinear PCS in Lasso and Jolt"
date: 2023-10-20T11:03:00+02:00
draft: false
mathjax: true
---


# Problem statement

In this write-up, we aim to provide context for the Need for Speed: why do we need efficient (multilinear) polynomial commitment schemes for lookups, specifically in the context of Lasso and Jolt. 

To put the problem into context, we first motivate the need for Lasso, and lookups in general, with an example. We then briefly describe the non-trivial marriage of lookup arguments with general SNARKs for R1CS. To complete the big picture, we provide a short introduction to Lasso’s solution to lookups - and where exactly the polynomial commitment schemes come up.

In a follow-up post, we will present the actual results benchmarking 4 PC schemes implemented with an arkworks backend: Ligero, Brakedown, Hyrax 
and KZG.

## Lasso overview

In summary, Lasso is a lookup argument (family). It lets the prover commit to a vector of $m$ field elements and prove that they reside in some predetermined table $t$ of $n$ elements. How should we think about this vector and table, though? What do they really represent?

### Motivation for lookups

First, we need to put Lasso, or lookup arguments in general, into some context. Suppose the prover has a long and complex program that he wishes to prove a correct execution of. A natural step would be to arithmetize this computation (e.g. using R1CS) and later use some off-the-shelf SNARK to succinctly generate a proof. The problem with this naive approach is that real-world computation rarely operates on finite field elements as required by R1CS - instead, the program makes use of CPU friendly types, such as 64-bits integer, to construct some complex computation.

This leads to a mismatch of representations and our arithmetization needs to account for this: otherwise the arithmetic circuit gives extra adversarial power to the prover by letting him “wrap around” the field modulus. Consider e.g. the simple program:

```latex
x^2 = y
```

Where `x` is the witness and `y` is the public instance. We convert it to a R1CS instance:

```latex
A = [0 1 0]; B = [0 1 0]; C = [0 0 1];    z = [1 x y]^T
```

The prover wants to convince the verifier that he knows a satisfying assignment to `z` such that $A\cdot z \odot B \cdot z = C \cdot z$ for a partially known `z = [1 x 25]`, say. By inspection we see that `x = 5`, but since we’re working with finite fields, there could be two solutions: the answer could also be $p - 5$, for a field modulus $p$. More precisely, a unique solution in the natural numbers (here, unsigned integers) is not "supported" with this relation over fields.

The way to circumvent this issue is to introduce restrictions on the bit decomposition of `x`: constrain every bit to be either 0 or 1 (duh!), and require the weighted sum of all bits by suitable powers of two to equal $x$. To constrain each bit `x_i` of the **field element** `x` we impose the restriction that $x_i^2 = x_i$. These two conditions asssure that $x$ fits in the prescribed number of bits (e.g. 64) and we have a unique solution.

In R1CS language, this would look something like:

```latex
z = 
[1  x  x_0 x_1 x_2 ... x_63   y]^T

A = 
[0  1  0   0   0  ...   0     0] // for x^2 = y
[0  0  1   0   0  ...   0     0] // for x_0^2 = x_0
[0  0  0   1   0  ...   0     0] // for x_1^2 = x_1
...
[0  0  0   0   0  ...   1     0] // for x_63^2 = x_64
[0  0  1   2   4  ...   2^63  0] // for x_0 + 2*x_1 + ...  2^63*x_63 = x

B = 
[0  1  0   0   0  ...   0     0] // for x^2 = y, same as A
[0  0  1   0   0  ...   0     0] // for x_0^2 = x_0, same as A
[0  0  0   1   0  ...   0     0] // for x_1^2 = x_1, same as A
...
[0  0  0   0   0  ...   1     0] // for x_63^2 = x_63, same as A
[1  0  0   0   0  ...   0     0] // for x_0 + 2*x_1 + ...  2^63*x_63 = x, different!

C = 
[0  0  0   0   0  ...   0     1] // for x^2 = y
[0  0  1   0   0  ...   0     0] // for x_0^2 = x_0
[0  0  0   1   0  ...   0     0] // for x_1^2 = x_1
...
[0  0  0   0   0  ...   1     0] // for x_63^2 = x_63
[0  1  0   0   0  ...   0     0] // for x_0 + 2*x_1 + ...  2^63*x_63 = x
```

We’ve now assured the verifier that we have a valid solution to `x^2 = y` AND that `x` is small (fits into 64 bits). This carries a massive overhead, though: for every element `x` in our program that is not naturally a field element, such as `u64`, we need to add $64 + 1$ additional constraints to convince the verifier of its bit decomposition.

Can we do better?

### Enter the lookup (table)

The motivating question is then: could we use lookups instead of bit decomposition?

Lookup arguments phrase the question as following:

Suppose we want to prove that a variable in the computation is within the range $[0, 2^{64})$. Can we show that it is one of the values in some publicly known table $t$, whose entries are all the integers between $0$ and $2^{64} - 1$?

There has been a line of work exploring different techniques as well as trade-offs for achieving that goal. These have led to lookup arguments: protocols for convincing the verifier that one or more committed elements reside in some publicly known table $t$.

However, it is not immediately clear how to combine a lookup argument with a general SNARK, since they represent fundamentally different relations. The statement proved in a SNARK is about arithmetic circuit satisfiability, whereas a prover in a lookup argument convinces the verifier that all elements in a committed vector reside in a public table.

We’ve seen some SNARKs utilizing the Plonk-ish arithmetization that have integrated a lookup into the relation, but it doesn’t seem to translate directly to SNARKs based on R1CS arithmetization. [This blog post](https://ethresear.ch/t/grolup-plookup-for-r1cs/14307) explores a potential idea borrowed from LegoSNARK for adding Plookup to Groth16.

The key insight there is that we can combine two different arguments (e.g. for different parts of our computation) if they have an explicit (and similarly structured) commitment to the same witness - which is not the case e.g. for Groth16, where the proof contains the committed witness only implicitly. 

In the case of Jolt, the commitment used in the lookup is actually a subset of the commitment used for proving general circuit satisfiability. Think back to the previous example where the witness consisted of 65 elements: `w = [x x_0 x_2 ... x_63]`: now by strapping a lookup onto our SNARK, we can forego the long witness and keep just `w = [x]`. Aside from a shorter witness, we have immediately reduced the overhead in terms of number of constraints by discarding all the binary decomposition checks. Instead, to prove that $x$ fits into 64 bits we use a lookup argument into a giant-but-highly-structured table of $2^64$ elements.

Note that in our trivial relation, the commitment to the vector used in the lookup, $x$, is the same as the commitment of the entire R1CS witness. In general this won't be the case and we would need to identify the subset of the R1CS witness that is going to be used in the lookup argument.

For example, for a witness $w$ of length $2n$ (for simplicity, assume $n$ is a power of two), imagine that only the first $n$ R1CS variables are part of the lookup argument. 

Without diverging into the details, we state that the SNARK employed here to reason about R1CS satisfiability is [Spartan](https://eprint.iacr.org/2019/550). The Spartan prover commits to the witness $w$ by a way of low-degree extending it and then committing to that polynomial.

With the perspective of adding a lookup argument, rather than committing to the MLE of the entire witness vector $\widetilde{w}$ directly, notice that:
$$\widetilde{w}(x_1, ..., x_{\log(n)+1}) = (1-x_1) \times \widetilde{w_1}(x_2, ..., x_{\log(n) + 1}) + x_1 \times \widetilde{w_2}(x_2, ..., x_{\log(n) + 1})$$

So instead of committing to $\widetilde{w}$, we can split the witness in two halves and commit to their MLEs separately: first to $\widetilde{w_1}(x_1, ..., x_{\log(n)})$, then to $\widetilde{w_2}(x_1, ..., x_{\log(n)})$. By the above relation, and for homomorphic commitments, the verifier can easily obtain the commitment to the full $w$ by himself, and later query the committed $w$ by asking for the opening of both $\widetilde{w_1}$ and $\widetilde{w_2}$ (at the same point) and combining the result.

This technique enables us to use one argument (e.g. SNARK) on the entirety of the committed witness and another argument (e.g. a lookup argument) on a subset of the committed elements - while maintaining a link between the two parts.

We have so far only given an intuition for where lookups would be useful and how to integrate them into an R1CS-arithmetized “multilinear SNARK”. We have intentionally skipped the details of how lookups actually work. In fact, Lasso mechanics differ substantially from most of the existing lookup arguments and so we will not cover these. For an excellent overview of the literature [“A Brief History of Lookup Arguments”](https://github.com/ingonyama-zk/papers/blob/main/lookups.pdf).

 

### Lasso in a nutshell

The Lasso verifier seeks to confirm that the relationship $M \cdot t = a$ holds, where:

- $t$ is a large lookup table with some “good” structure, of size $N$,
- $a$ is the vector of elements to be looked up. If we want to perform $m$ lookups in our protocol, then $a$ has size $m$.
- $M$ is the $m \times N$ matrix whose rows are unit vectors in such a way as to make the above true if and only if all entries of $a$ are contained in $t$. Such a matrix always has exactly one non-zero entry per row. Indeed, if the i-th entry of $t$ is the j-th element of $a$, then the j-th row of $M$ is a one-hot vector concentrated in its i-th entry.

To give a tiny example, set: $m  = 2$, $N = 4$. The prover wants to show that all elements of $a = (1, 3)$ are within the range $[0, 4)$ - i.e. they are contained in the table $(0, 1, 2, 3)$.

Then a satisfying assignment would be:

$$\begin{pmatrix}   0 & 1 & 0 & 0 \\\   0 & 0 & 0 & 1\end{pmatrix} \cdot \begin{pmatrix}   0 \\\ 1 \\\ 2 \\\ 3\end{pmatrix} = \begin{pmatrix}   1 \\\ 3 \end{pmatrix} $$

Typically, in our succinct protocol the (low-degree extensions of the) matrices $a$ and $M$ are committed to and the verifier is only given oracle access to them, with the oracle instantiated via a polynomial commitment scheme. We assume that $t$ has some “nice” structural properties to allow the verifier to efficiently work with it, as described later.

Unlike in our tiny example, $m$ is often **much** smaller than $N$: $m \ll N$.

So committing to $a$ can be feasible, but notice that the number of entries of $M$ is actually larger by a factor of $m$ than the number of entries of the lookup table. If we want to use humongous lookup tables in our protocol (think $2^{128}$) then we certainly can’t allow ourselves to commit to that many elements of $M$.

Instead, we leverage the following fact: most of the entries in $M$ are $0$ (the matrix is sparse). More precisely, we won’t be committing to a matrix per se - rather, to its multilinear extension (MLE) (for more details on how to transform any function to low-degree extensions, see Sec. 3.5 of the [Proofs, Args and Zero-Knowledge](https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf) survey by J. Thaler). Furthermore, rather than checking that the statement $M \cdot t = a$ holds for all entries of $a$, we pick a random point $r \in \mathbb{F}^{\log(m)}$ and, if the equality below holds for $r$, then the original statement holds with high probability:

$$
\sum_{j \in \\{0,1\\}^{\log(N)}} \widetilde {M}(r, j) \cdot \widetilde t(j) = \widetilde{a}(r),
$$

where $M(x, y)$ is the function that maps indices x, y, represented as bits, to the entry $M_{x, y}$.

Finally, a quick note on the table $t$: we assume that $t$ has enough structure for the verifier to efficiently evaluate it at any single point. Since the above equation is begging to be sumchecked, at the conclusion of the sumcheck protocol we require the verifier to only query the MLE of the table $\widetilde{t}$ at one single point $r’$. Details of how $t$’s structure lends itself to efficient evaluation are not necessary for our objective here and can be found directly in the Lasso paper.

Therefore, we have established that in order to prove membership of all elements of $a$ in a lookup table $t$, Lasso needs a way to efficiently commit to a very large-but-sparse matrix $M$.

### IDK how it works, but I know why it’s expensive,

…or glossing over Spark & Surge.

Spark is a way to commit to sparse multilinear polynomials with the prover costs proportional to the number of non-zero evaluation over the boolean hypercube, rather than to the size of the hypercube itself, as would be the case for dense polynomials. The approach of Spark is to commit to such a sparse vector of evaluations, and later prove the correct computation of its dot product with the Lagrange basis evaluated at the random point queried by the verifier.

We view any multilinear polynomial’s $\widetilde f$ over $v$ variables as:

$$
\widetilde{f}(x_1, ..., x_v) = \sum_{w \in \\{0,1\\}^v} f(w) \cdot \chi_w(x_1, ..., x_v)
$$

With $f(w)$ being the evaluations over the Boolean hypercube, and $\chi_w(x_1,…x_v)$ the multilinear Lagrange basis polynomial corresponding to $w$.

We are already given $f(w)$ as part of the description of the multilinear polynomial.

Therefore, if we can somehow evaluate $\chi_w(r_1,…r_v)$ efficiently for all $w \in \\{0,1\\}^v$, then we are in a good position to compute the dot product. It turns out that exists an efficient procedure to do this.

The authors of Lasso notice that Spark can be generalized to prove inner products for other purposes than just opening a polynomial at a random point.

Surge is such a generalization, which replaces the Lagrange basis in above equation with any function efficiently evaluateable at any $w$, specifically with the function $t(w)$ that maps any index $w$ to an element of our table.

The way that Spark and Surge prove this relationship holds is by leveraging an adaptation of a technique called offline memory checking. Here is where we disappoint the reader and instead refer them to Sec. 5 of Lasso, where the scheme is described in a very accessible manner.

Instead, we simply summarize the costs in the full realization of offline memory checking when applied to Lasso. Since “pure” field operations are at least an order of magnitude faster than cryptographic operations involved in committing to the polynomials, we omit the former.

************Prover:************

- Commit to $c$ polynomials over $\log(m)$ variables
- Commit to $2\cdot \alpha$  polynomials over $\log(m)$ variables
- Commit to $\alpha$ polynomials over $\log(N^{1/c})$ variables
- Prove openings of $\alpha$  $\log(m)$-variate polynomials, all at one point $r_z$
- Prove openings of $3$  $\log(m)$-variate and one $\log(N^{1/c})$-variate polynomials, all at one point $r_b$ (after batching sumchecks)

******************Verifier****************** 

- Verify the corresponding openings

Here $c$ is a parameter we choose freely and $\alpha = c \cdot k$ for a small constant $k$. We recall that $N$ is the size of the table and $m$ is the number of lookups.

Since our objective is to pick the best candidate for Jolt, let’s focus on its flagship application, emulation of RISC-V ISA over the 64-bit data types. A reasonable parameter choice there is $c = 6$ and $N = 2^{128}$. The bottleneck for the prover is then committing to polynomials over ~20 variables. This is what we target in our benchmarks.

We also note that most of the committed elements are “small”, in the sense that they can be represented by field elements of size, say $\lt 2^{64}$. Some PC schemes can take advantage of this, including KZG and Hyrax, where the critical component of the scheme is the multi-scalar exponentiation, which naturally favors smaller exponents.

Armed with all this knowledge, we are now ready to pick the best PCS candidate. Almost.

## What should I aim for?

As with everything, it depends. Some possible targets are:

- Prover time
- Verifier time
- Proof size
- On-chain verification
- Recursion friendliness

Context choice will play an important role here. For the use cases where we care about on-chain verification, we need to be mindful of gas costs associated with the verification, as well as proof sizes that we upload. Optimizing for verifier speed will often incur additional prover complexity, which might become prohibitive for large computation. For big enough programs, proof composition might bridge this gap.

In the next post we will explore the different tradeoffs. We will present concrete results for native prover-verifier runtimes, as well as benchmarks and estimates for proof composition with the verifier encoded as a halo2 circuit.


---
Author: Marcin Górny

Thank you Antonio Mejías Gil & Giacomo Fenzi for helpful comments and discussions.
