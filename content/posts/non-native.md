---
title: "Non-native arithmetic"
date: 2022-08-09T11:03:00+02:00
draft: true
math: true
---

As with the previous post, this isn't intended to serve as an introduction to SNARKs, recursive proofs nor pairing-friendly curves. Rather, it's an attempt to explain a topic that comes up often in conjunction with the aforementioned terms, namely **non-native field arithmetic**. I'll first explain the *why*: the context of where non-native arithmetic arises. Then I will try to define the *what*: by comparing it to native arithmetic and describing in what part of SNARKs each of them is used. Finally, I'll close with a brief summary of the techniques used for realising non-native arithmetic efficiently.

# Non-native arithmetic: the *why*

The notion of native vs. non-native arithmetic doesn't really arise in a non-recursive setting, such as when a prover sends a proof that encodes the entire circuit in a "flat" proof, and the verifier performs some pairing checks to either accept or reject the proof. 

The environment where the prover algorithm runs is completely separated from the environment of the verifier. In fact, they are likely to be separate machines altogether.
<!-- 
In contrast the verifier, in order to compute a pairing, performs arithmetic over the base field: e.g. the algorithm for the Miller loop, a key component in pairings, operates on rational functions $f(x,y)$ where $x,y \in 𝔽_q$, so clearly the pairing arithmetic must be done over the base field. -->

## Recursion

So far so good, because the parties are completely separate and don't need to care about one another's arithmetic preferences. The challenge arises when we wish to compose the proofs via recursion. For a variety of applications such as zk-rollups or incrementally verifiable computation, we might want to split the proof generation into smaller, more managable parts. An intuition would be that if the prover's compute and memory requirements are linearly dependant on the statement size, then for very large statements which might be computationally infeasible under direct construction, the computation is split into smaller statements and recursed. For example, rather than proving that a block in the Bitcoin network is valid by directly proving all of the state transitions since genesis in one go, we instead prove that given the previous block and the current state transition, the new block is indeed valid, and recurse back to genesis.

The overall idea of proof recursion is that the circuit doesn't encode the full computation (all state transitions), but rather a small part of it (single state transition), plus an additional circuit to represent the verifier *which runs inside the prover's circuit*. This way, a proof of the validity of some final state $S_n$ contains a proof for that final transition from $S_{n-1} -> S_n$ was computed correctly, as well as a proof that the transition to $S_{n-1}$ was valid. Of course this only makes sense if |circuit for encoding the verifier| + |circuit for the sub-statement| < |circuit for the entire statement|[^1].

[^1]: And if the verifier's computation inside the circuit doesn't increase in size as we recurse.


Here's the caveat: the arithmetic of the prover and the arithmetic of the verifier are performed over two different finite fields. If the prover wishes to encode the logic for the verifier within their circuit constraints, they must be able to do arithmetic over the (verifier's native, prover's non-native) $𝔽_q$, rather than (prover's native) $𝔽_r$.

If this all sounds a bit abstract, let's walk through some of the operations that each party performs.

# Non-native arithmetic: the *what*

To answer what non-native arithmetic is, perhaps one should start with the definition of "native" arithmetic.

## Base field

Elliptic curves useful for cryptographic applications are usually defined over a finite field, $𝔽_q$ - usually called the base field, where $q$ is prime power ($q = p^k$ for prime $p$).

This means that each point on the curve $E(𝔽_q)$ has its coordinates constrained to lie within the range $[0,q)$.

E.g. let's take a field with $q = 47$, and a curve equation $E(𝔽_q): y^2 = x^3 + 5x$
A valid point on the curve is one which satisfies the curve equation. 
Some valid points are:
* (28, 7)

But there are many invalid points too:
* (28, 8)

The main point here is that all arithmetic done on the curve requires reductions modulo $q$. Determining whether a point lies on the curve is just one example of such a computation.
Imagine another application, where (for reasons that will be explained later) we are given a rational function $f: E(𝔽_q) \to 𝔽_q$ which takes as input a point on the curve $P$ and operates on its coordinates $x, y$ to output a field element. It is meaningless to perform arithmetic on coordinates $x, y \in 𝔽_q$ over a modulus different than $q$.

### Base field in pairings

I will make a big leap here and jump right from the definition of the base field to elliptic curve pairings. Don't worry though, we will neither cover the formal definition nor all the details. In order to understand where non-native arithmetic is used in SNARKs, it is enough to accept the following:
- verifier must perform a pairing check $e(\cdot, \cdot) \stackrel{?}{=} e(\cdot, \cdot)$ to convince themselves of the validity of some statement
- the inputs to a pairing $e$ are group elements belonging to two (potentially distinct) cryptographic groups $G_1$ and $G_2$
- they are the subgroups of some $E(𝔽_{q^k})$ (note that $k$ might be 1 for one or both of the groups)
- arithmetic on $E(𝔽_{q^k})$ is done over the field $𝔽_q$ no matter the embedding degree $k$
- the key subroutine of all pairings is a Miller loop: it works by iteratively applying some rational function and accumulating the result as a field element
- the rational functions in the Miller loop are straight lines:
    - their equation depends on the first input $P$ (from $G_1$). They define the intersection of the a line through $P$ and some multiple of $P$, with the curve $E(𝔽_{q^k})$.
    - they are evaluated at the coordinates of the second input element $Q$ (from $G_2$).

It naturally follows that the arithmetic in the iterations of the Miller loop are performed over the base field $𝔽_q$. Thus, the verifier's native field is the base field $𝔽_q$ of the curve $E(𝔽_q)$.

## Scalar field

Before defining the scalar field, we need to define one more concept: groups.
It would be great if each set of valid elliptic curve points automatically formed a group.
Without diving into the formalism, let's state the practical properties of the group $G$ that we care about here:
* some element of this group can generate all other group members via repeated addition of itself, call it the generator $g$
* there exists an integer (scalar) $r$, s.t. for any $g \in G$, $r \cdot g = 1$ and $r$ is prime. Call it the order of $G$.
* it is cyclic (as a result of the previous two points)

So what we are after is usually referred to as a subgroup $G$ of $E(𝔽_q)$ of prime order $r$. When defining curves for PBC, we usually specify the scalar field $𝔽_r$ (alongside the previously mentioned $𝔽_q$).
In practice, when working with SNARKs the data that is passed between the parties (such as the prover and the verifier) will be encoded as group elements, and the circuit representing our computation will be defined over the scalar field $𝔽_r$.

## Scalar field in circuits

[Recall](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649) that in order to represent an arithmetic circuit corresponding to the statement that we wish to prove, we need to represent the circuit as a set of polynomials.

Ultimately, we would like to be able to commit to such polynomials and open them at certain points as challenged by the verifier. Interestingely, the variable of our (univariate) polynomial will assume en element of a finite field AND an element of a group.

Specifically, when we commit to a polynomial, we will output a group element, and when we open a polynomial, we will output both a field element and a group element. What really matters though, is that the order of both is $r$, mentioned in the previous section.

This boils down to the coefficients of our polynomial being in the range $\[0, r\)$, and so all arithmetic is performed mod $r$. Notice that it wouldn't be of much use to multiply a group element by something bigger than $r$, say $r+5$, since $r \cdot g = 1$, so $(r+5) \cdot g = 5 \cdot g$.

In a typical instantiation of a polynomial commitment scheme, it is the prover's job to caluclate the commitment of their secret polynomial, then evaluate the polynomial at a concrete challenge $z$ and send the claimed evaluation $y = p(z)$ to the verifier (together with the proof).

With this slightly informal argument we establish that the prover needs to perform work over the scalar field.

# Non-native arithmetic: the *how*

The idea of this post was to give the reader an intuition of what the terms "native" and "non-native" arithmetic mean in the context of SNARKs, without going into how they are realised in practice. It would be incomplete, however, to completely skip it, so here's my best attempt at summarising it in a few sentences.

To achieve recursion, we could try to simulate the non-native arithmetic inside a circuit, as a native computation. This would require splitting the large mod-q elements into smaller limbs, s.t. computation can be safely performed with the small mod-r, without the danger of "overflowing". This, unfortunately, imposes a lot of overhead on the circuit and as such is inefficient in practice.

An alternative approach, pioneered in [BCTV14](https://eprint.iacr.org/2014/595.pdf), was the use of cycles of pairing-friendly curves[^2]. The idea is that instead of having a SNARK defined over a single curve, we have two SNARKs, each defined over a separate curve, s.t. the base field of one curve is the scalar field of the other, and vice-versa. This allows for representing the verifier's circuit efficiently (without the overhead of simulating mod-q arithmetic with mod-r arithmetic) and the resulting protocol alternates between calling the two SNARKs.

[^2]: To find out what makes a curve pairing-friendly, I refer the interested reader to [Craig Costello's excellent guide to pairings](https://www.craigcostello.com.au/s/PairingsForBeginners.pdf) or [Ben Lynn's dissertation](https://crypto.stanford.edu/pbc/thesis.pdf).

# Outlook

The advent of recent constructions such as accumulator schemes (see [here](https://eprint.iacr.org/2019/1021.pdf) or [here](https://eprint.iacr.org/2020/499.pdf)) or [folding schemes](https://eprint.iacr.org/2021/370) for achieving incrementally verifiable computation might render the notion of "(non-) native" arithmetic somewhat obsolete. While there still exists a need to encode the verifier' computation inside the prover's circuit, the scope of verifier's work during recursion can be greatly reduced from SNARK verification to the mentioned accumulator/folding schemes. As a result, the most expensive part of that circuit (the pairing check) disappears [^3]. 

More on accumulator schemes another time.

[^3]: There is still a pairing to be done, but it only happens once at the end of the protocol, outside of the recursion loop.
