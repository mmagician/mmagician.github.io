---
title: "Non-native arithmetic"
date: 2022-08-09T11:03:00+02:00
draft: true
math: true
---

As with the previous post, this isn't intended to serve as an introduction to SNARKs, recursive proofs nor pairing-friendly curve cycles. Rather, it's an attempt to explain a topic that comes up often in conjunction with the aforementioned terms, namely the usage of non-native field arithmetic. I'll first explain the *why*: the applications and settings where non-native arithmetic is used. Then I will try to define what it is in the context of comparing it to native arithmetic, and explain some basic math that helps to understand how both types arise. Finally, I'll close with a small summary of the techniques used for implementing non-native arithmetic in practice.

# Non-native arithmetic: the *why*

The notion of native vs. non-native arithmetic doesn't really arise in a non-recursive setting, such as when a prover sends a proof that encodes the entire circuit in a "flat" proof, and the verifier performs some pairing checks to either accept or reject the proof. 

In this setting, the prover creates the proof by ..., 

In contrast the verifier, in order to compute a pairing, performs arithmetic over the base field: e.g. the algorithm for the Miller loop, a key component in pairings, operates on rational functions $f(x,y)$ where $x,y \in 𝔽_q$, so clearly the pairing arithmetic must be done over the base field.

## Recursion

So far so good, because the parties are completely separate and don't need to care about one another's arithmetic preferences. The challenge arises when we wish to compose the proofs via recursion. For a variety of applications such as zk-rollups or incrementally verifiable computation, we might want to split the proof generation into smaller, more managable parts. An intuition would be that since the prover's work and memory requirements are linearly dependant on the statement size, then for very large statements which might be computationally infeasible under direct construction, the computation is split into smaller statements and recursed.

The overall idea of proof recursion is that the circuit doesn't encode the full computation, but rather a small part of it, plus an additional circuit to represent the verifier *which runs inside the prover's circuit*. This way, a proof of the validity of some final state $S_n$ contains a proof for that final transition from $S_{n-1} -> S_n$, as well as a proof that the transition to $S_{n-1}$ was valid. Of course this only makes sense if |circuit for encoding the verifier| + |circuit for the sub-statement| < |circuit for the entire statement|.

However, as we've just learned, the arithmetic of the prover and the arithmetic of the verifier are performed over two different finite fields. If the prover wishes to encode the logic for the verifier within their circuit constraints, they must be able to do arithmetic over the (verifier's native, prover's non-native) $F_q$.

If this all sounds a bit abstract, let me walk you through some of the operations that each party performs.

# Non-native arithmetic: the *what*

To answer what non-native arithmetic is, perhaps one should start with the definition of "native" arithmetic.

## Base field

Elliptic curves useful for cryptographic applications are usually defined over a finite field, $𝔽_q$ - usually called the base field, where $q$ is prime power ($q = p^k$ for prime $p$).
This means that each point on the curve $E(𝔽_q)$ has its coordinates constrained to lie within the range $[0,q)$.

E.g. let's take a field with $q = 47$, and a curve equation $E(𝔽_q): y^2 = x^3 + 5x$
A valid point on the curve is one which satisfies the curve equation. 
Some valid points are:
* (4, 18)

But there are many invalid points too:
* ()

The main point here is that all arithmetic done on the curve requires reductions modulo $q$. Determining whether a point lies on the curve is just one example of such a computation.
Imagine another application, where (for reasons that will be explained later) we are given a rational function $f: E(𝔽_q) \to 𝔽_q$ which takes as input a point on the curve $P$ and operates on its coordinates $x, y$ to output a field element. It is meaningless to perform arithmetic on coordinates $x, y \in 𝔽_q$ over a modulus different than $q$.

## Excerpt from pairings

I will make a big leap here and jump right from the definition of the base field to elliptic curve pairings. Don't worry though, we will neither cover the formal definition nor all the details. In order to understand where non-native arithmetic is used in SNARKs, it is enough to state the following:
- verifier must perform a pairing check $e(\cdot, \cdot) \stackrel{?}{=} e(\cdot, \cdot)$ to convince themselves of the validity of some statement
- the inputs to a pairing $e$ are group elements belonging to two (potentially distinct) cryptographic groups $G_1$ and $G_2$
- they are the subgroups of some $E(𝔽_{q^k})$ (note that $k$ might be 1 for one or both of the groups)
- arithmetic on $E(𝔽_{q^k})$ is done over the field $𝔽_q$ no matter the embedding degree $k$
- the key subroutine of all pairings is a Miller loop: it works by iteratively applying some rational function and accumulating the result as a field element
- the rational functions in the Miller loop are straight lines:
    - depending on the first input from $G_1$, s.t. they intersecting with the curve at multiples of the input
    - the obtained line equations are evaluated at the coordinates of the second input element from $G_2$

It naturally follows that the arithmetic in the iterations of the Miller loop are performed over the base field $𝔽_q$. Thus, the verifier's native field is the base field $𝔽_q$ of the curve $E(𝔽_q)$.

## Scalar field

Before defining the scalar field, we need to define one more concept: groups.
It would be great if each set of valid elliptic curve points automatically formed a group. 
Without diving into the formalism, let's state the practical properties of the group $G$ that we care about here:
* some element of this group can generate all other group members via repeated addition of itself, call it the generator $g$
* there exists an integer (scalar) $r$, s.t. $r \cdot g = 1$ and $r$ is prime. Call it the order of $G$.
* it is cyclic (as a result of the previous two points)

So what we are after is usually referred to as a subgroup $G$ of $E(𝔽_q)$ of prime order $r$.
In practice when working with SNARKs (and a lot of other ECC constructions), the data passed between the parties (such as the prover and the verifier) will be encoded as group elements.

Specifically, this is how the circuit will be represented.

## Circuit representation

[Recall](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649) that in order to represent an arithmetic circuit corresponding to the statement that we wish to prove, we need to represent the circuit as a set of polynomials.




# Non-native arithmetic: the *why*

# Non-native arithmetic: the *how*

Cycles of pairing-friendly curves.

To find out what makes a curve pairing friendly, I refer the interested reader to [Craig Costello's excellent guide to pairings]() or [Ben Lynn's dissertation]().

The first to realise an efficient construction for recursive proof composition via cycles of curves were [BCTV14](https://eprint.iacr.org/2014/595.pdf). Previous attempts included simulating the non-native field within the native field, e.g. by splitting each non-native element into small limbs, s.t. the result of multiplying any two limbs fits within the size of the native field without reductions.


