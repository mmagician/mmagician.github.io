---
title: "Benchmarking multilinear Polynomial Commitment Schemes"
date: 2023-10-20T11:03:00+02:00
draft: true
mathjax: true
---

## Acknowledgments

This work would not have been possible without an outstanding effort from my team in the last 2 weeks. Huge thank you to Antonio for implementing Hyrax and to Hossein for developing Brakedown - you guys are true wizards!

# Previously

In the previous post we gave the intuition of why efficient commitments to multilinear polynomials are important in the context of lookups in Lasso. We aim for this entry to be self-contained, although the previous write-up provides useful context and a bigger picture.

## This post

Here, we present the actual results of our technical contribution: benchmarks and implementation of 4 PC schemes:

- Ligero: native & recursive implementation; benches,
- Brakedown: native implementation; benches,
- Hyrax: native implementation; benches,
- multilinear KZG: benches.

To our knowledge, to date there is no single library that implements all the above schemes with the same backend. There exist some academic implementations of e.g. Hyrax in python; multilinear KZG with the arkworks backend in Rust; Ligero/Brakedown with ZCash backend in Rust. As such, comparing the existing implementations for performance is hardly meaningful.

Our core contribution is implementing the first 3 schemes from scratch in arkworks - and adding benchmarks to the existing KZG module (from EspressoSystems/jellyfish, but also built with the arkworks backend). We also implement a halo2 verifier of Ligero to allow for potential on-chain verification.

## What should I aim for?

As hinted at in the previous post, our end goal should drive our design choices. We need to know what the constraints on our resources are. As such, some objectives we might wish to consider:

- Prover time
- Verifier time
- Proof size
- On-chain verification
- Recursion friendliness

## Native benchmarks

Out of the 4 chosen schemes, 3 are by nature interactive. The standard way to render such schemes non-interactive is to apply the Fiat-Shamir transform, which in practice means instantiating these with a cryptographic hash based sponge. Hashing in-circuit (needed for recursive verification) is a few orders of magnitudes slower for hashes based on bitwise-operations such as SHA256, than for algebraic hash functions such as Poseidon.

Therefore, for Ligero we consider 2 variants: optimized native with SHA256-based hashing & recursion support (R) with Poseidon. Multilinear KZG is non-interactive to start with, so no such transformation is necessary. The results for Ligero are strong enough to let us draw some conclusions for non-interactive Brakedown and Hyrax.

Furthermore, as noted above, for the specific application to Jolt where the committed elements are “small”, we will consider an additional variant of Hyrax using a modified implementation of MSM optimized for <60 bits. We call this variant small (S) to differentiate from the case of arbitrary (A) coefficients. 

When benchmarking the schemes, we noticed a curiosity: Ligero proofs were larger than Brakedown proofs! Give the same arrangement of polynomial coefficients into the matrix, one would naturally expect Brakedown to have larger proofs, due to more columns being opened. 

We then realized that our implementation of Ligero was always producing square matrices, while Brakedown’s were wider in columns and shorter in rows. We have since added an alternative implementation of Ligero+ to use the same arrangement as Brakedown.

We finally describe the last variation on Brakedown. “Plain” Brakedown is implemented with the 3rd row of parameters from [Figure 2 of the Brakedown paper](https://eprint.iacr.org/2021/1043) (small distance), while Brakedown+  has a bigger code distance and smaller `t` (number of column openings), but pays the price in prover time.

This gives us a total of 8 schemes:

- KZG
- Hyrax(S)
- Hyrax(A)
- Ligero
- Ligero+
- Ligero (R)
- Brakedown
- Brakedown+

We start with polynomials that have their coefficients in the scalar field of the BN254 curve. We fix this particular curve to support potential verification on Ethereum. 

The security of KZG and Hyrax is bound to the choice of the elliptic curve over which we perform group operations.

Our Ligero and Brakedown implementations are parametrized with the security parameter. We aim for at least of 128-bits of security for these schemes.

We note that while aiming for 128-bits of security for the linear-code based schemes, instead of e.g. 110 on BN254 for a more fair comparison, the performance difference from opening a slightly fewer columns is only a few percentage points. We intentionally skip these to contain our parameter space.

Our codebase for Ligero, Brakedown and Hyrax is hosted at: https://github.com/HungryCatsStudio/poly-commit.

To run the all benchmarks, use: `cargo bench --bench="pcs" --features="benches"`

Our fork of Espresso Systems’ Jellyfish library used for KZG is: https://github.com/HungryCatsStudio/jellyfish

To run the KZG benchmarks, run: `cargo bench --bench="pcs" --features="test-srs"`

We will be contributing all our schemes and benches to `arkworks/poly-commit` and `EspressoSystems/jellyfish` within the coming weeks as we complete minor refactors.

### Objective 1: Prover compute cost

All values given in milliseconds.

**Commit time; Linear-code based schemes**

$$
\begin{array}{|l|c|c|c|c|c|}
\hline
\text{Scheme \textbackslash Variables} & 12 & 14 & 16 & 18 & 20 \\\
\hline
\text{Ligero} & 6.73 & 19.92 & 65.68 & 202.91 & 630.88 \\\
\hline
\text{Ligero+} & 3.43 & 10.35 & 30.97 & 105.77 & 364.21 \\\
\hline
\text{Ligero(R)} & 139.34 & 533.48 & 2056 & 8236 & - \\\
\hline
\text{Brakedown} & 3.67 & 11.68 & 39.84 & 144.71 & 615.33 \\\
\hline
\text{Brakedown+} & 4.71 & 15.12 & 52.98 & 194.86 & 829.41 \\\
\hline
\end{array}
$$

************************Commit time; KZG; Hyrax (A); Hyrax(S); Linear-code winner************************

$$
\begin{array}{|l|c|c|c|c|c|}
\hline
\text{Scheme \textbackslash Variables} & 12 & 14 & 16 & 18 & 20 \\\
\hline
\text{Hyrax(A)} & 14.315 & 40.087 & 98.262 & 302.44 & 1425 \\\
\hline
\text{Hyrax (S)} & 8.65  & 35.62 & 102.28 & 324.48  & 1010  \\\
\hline
\text{KZG} & 4.6724 & 15.202 & 49.417 & 158.92 & 562.01 \\\
\hline
\text{Ligero+} & 3.43 & 10.35 & 30.97 & 105.77 & 364.21 \\\
\hline
\end{array}
$$

With **Ligero+** winning not only among the linear-code-based schemes, but all schemes we tested in terms of commit time. This seems to be in line with the benchmarks from the Brakedown paper. Brakedown has asymptotically linear time, but either the implementations so far haven’t been able to capture it efficiently enough, or the extra $\log(n)$ costs of Ligero are still outperforming Brakedown concretely until very large $n$ (not seen here).

Ligero(R) performs about significantly slower than Ligero for committing (and open & verify), as expected.

**********Open; Linear-code based schemes**********

$$
\begin{array}{|l|c|c|c|c|c|}
\hline
\text{Scheme \textbackslash Variables} & 12 & 14 & 16 & 18 & 20 \\\
\hline
\text{Ligero} & 16.33 & 44.64 & 140.58 & 431.89 & 1355.7 \\\
\hline
\text{Ligero+} & 20.43 & 48.52 & 119.72 & 334.62 & 1076.0 \\\
\hline
\text{Ligero(R)} & 287.94 & 1081 & 4211 & 16448 & - \\\
\hline
\text{Brakedown} & 49.69 & 120.85 & 293.22 & 732.71 & 2172.3 \\\
\hline
\text{Brakedown+} & 54.33 & 138.63 & 341.35 & 886.66 & 2637.8 \\\
\hline
\end{array}
$$

We notice that our implementations for linear code schemes ALL miss out on the optimization in the open phase, where the prover shouldn’t need to re-hash all the columns, since they’ve already done in the commit phase. Especially natively and when using an efficient bitwise-based hash, this might not be a large overhead, but still leaves room for improvement.

******************************************************************Open; KZG; Hyrax (A); Hyrax(S); Linear-code winner******************************************************************

$$
\begin{array}{|l|c|c|c|c|c|}\hline\text{Scheme \textbackslash Variables} & 12 & 14 & 16 & 18 & 20 \\\
\hline
\text{Hyrax(A)} & 16.305 & 44.048 & 105.66 & 318.15 & 1528.5 \\\
\hline
\text{Hyrax(S)} & 10.79 & 36.06 & 115.22 & 339.57 & 1080.1 \\\
\hline
\text{KZG} & 5.0280 & 19.815 & 72.308 & 221.85 & 712.02 \\\
\hline
\text{Ligero+} & 20.43 & 48.52 & 119.72 & 334.62 & 1076.0 \\\
\hline\end{array}
$$

For opening, the competition is more even and **multilinear KZG** emerges victorious here. One factor that might have skewed the scale is that the KZG implementation from jellyfish library is much more mature than our schemes, built in the last 2 weeks.

### Objective 2: Verifier compute cost

******************************************************************Verify time; linear-code based schemes******************************************************************

$$
\begin{array}{|l|c|c|c|c|c|}
\hline
\text{Scheme \textbackslash Variables} & 12 & 14 & 16 & 18 & 20 \\\
\hline
\text{Ligero} & 21.73 & 59.13 & 163.67 & 471.99 & 1441.4 \\\
\hline
\text{Ligero+} & 30.15 & 59.69 & 133.57 & 363.21 & 1142.3 \\\
\hline
\text{Ligero(R)} & 419.44 & 1307 & 4631 & 17324 & - \\\
\hline
\text{Brakedown} & 115.21 & 231.19 & 430.08 & 896.32 & 2409.6 \\\
\hline
\text{Brakedown+} & 116.17 & 217.71 & 437.97 & 985.24 & 2812.4 \\\
\hline
\end{array}
$$

************************Verify; KZG; Hyrax (A); Hyrax(S); Linear-code winner************************

$$
\begin{array}{|l|c|c|c|c|c|}
\hline
\text{Scheme \textbackslash Variables} & 12 & 14 & 16 & 18 & 20 \\\
\hline
\text{Hyrax(A)} & 17.539 & 45.826 & 110.50 & 327.92 & 1548.7 \\\
\hline
\text{Hyrax(S)} & 11.77 & 40.47 & 113.32 & 354.60 & 1114 \\\
\hline
\text{KZG} & 14.401 & 39.779 & 129.37 & 350.15 & 1204.5 \\\
\hline
\text{Ligero+} & 30.15 & 59.69 & 133.57 & 363.21 & 1142.3 \\\
\hline
\end{array}
$$

Ligero family is the expected winner in the code-based schemes due to small number of column openings thanks to its large code distance. The code distance in Brakedown requires opening many more columns for the comparable level of security.

Regarding the other schemes, the difference is so small that given our unoptimized implementation it’s hard to pinpoint the true winner, although for large enough instance sizes, **Hyrax(S)** seems to take the lead.

### Objective 3.1: Proof size, in bytes

$$
\begin{array}{|l|c|c|c|c|c|}
\hline
\text{Scheme \textbackslash Variables} & 12 & 14 & 16 & 18 & 20 \\\
\hline
\text{Ligero} & 305201 & 1144881 & 2683569 & 5260105 & 10400737 \\\
\hline
\text{Ligero+} & 244297 & 369121 & 606329 & 1068305 & 1979817 \\\
\hline
\text{Brakedown} & 1901009 & 3229073 & 4232905 & 6064001 & 9549721 \\\
\hline
\text{Brakedown+} & 1555665 & 1947569 & 2631025 & 3897713 & 6330705 \\\
\hline
\text{Hyrax} & 2360 & 4408 & 8504 & 16696 & 33080 \\\
\hline
\text{KZG} & 776 & 904 & 1032 & 1160 & 1288 \\\
\hline
\end{array}
$$

As we noted earlier, square-matrix Ligero has larger proofs than any Brakedown, but already Ligero+ outperforms both Brakedown variants as expected.

Nevertheless, schemes involving elliptic curve operations (KZG & Hyrax) beat the rest by orders of magnitude, with the clear winner being **KZG**.

### Objective 3.2: Commitment size

While not explicitly stated in the problem definition, we believe it is important to take the proof size into account. Hyrax is the only scheme with the commitment size dependent on the size of the polynomial. For the rest, this is a small constant: in the case of linear-code based PCS, it’s the size of the hash digest (in our case SHA256); in the case of KZG, a single group element, which for BN254 happens to also fit into 64 bytes.

$$
\begin{array}{|l|c|c|c|c|c|c|c|}\hline
\text{Scheme \textbackslash Variables} & 10 & 12 & 14 & 16 & 18 & 20 & 22 \\\
\hline
\text{Hyrax} & 2056 & 4104 & 8200 & 16392 & 32776 & 65544 & 131080 \\\
\hline
\text{Ligero} & 64 & 64 & 64 & 64 & 64 & 64 & 64 \\\
\hline
\text{Brakedown} & 64 & 64 & 64 & 64 & 64 & 64 & 64 \\\
\hline
\text{KZG} & 64 & 64 & 64 & 64 & 64 & 64 & 64 \\\
\hline
\end{array}
$$

## Recursive benchmarks

When we say that we are using SNARK recursion or composition, we refer to having the verifier of one (inner) scheme represented as a circuit of another (outer) scheme. We first run the prover for the inner circuit to obtain a valid (inner) proof. We then supply this proof as a (usually private) input to the outer circuit, and finally run the prover of the outer circuit, which implicitly means running an accepting inner verifier. This might seem a bit counterproductive, but there can be good reasons for this:

- We have access to a scheme with a fast prover, but large proof sizes. Our requirement is that proof sizes are small, though. We can “wrap” the fast-prover-scheme in a short-proof-size scheme.
- We have specific requirements on the verifier logic, e.g. we have an on-chain verifier that only accepts Groth16 proofs.

Now, however, in order to convince the outer-verifier of some statement, we have to run both the inner and the outer prover. We have to carefully consider when the balance tips in our favor when choosing such an approach.

Note that for recursive verification, we don’t necessarily care about the “native” verifier time, as the inner-verifier is in a way subsumed into the outer prover.

For the recursive benchmarks, we currently only have the empirical results for one scheme, Ligero, which we have implemented using Axiom’s `halo2-lib`. At the moment, this scheme remains the IP of our friends at Modulus Labs, for whom we’ve implemented the protocol. They will soon be open-sourcing the codebase - we ask to be trusted on the results for the time being. Modulus Labs have kindly agreed to let us use the code for benchmarking.

We extrapolate the number of columns that need to be opened in Ligero to the number opened in Brakedown. We provider very rough estimate for KZG and Hyrax based on the BN254 MSM benchmarks provided in https://github.com/axiom-crypto/halo2-lib#bn254-msm.

### Objective 4.1: Recursive prover compute cost

This is basically the outer-prover costs described above. When implementing schemes for recursive verification, one needs to take into account the fact that the inner-prover, adapted for recursion-friendliness, pays a large overhead in foregoing the efficient hash functions like SHA256 in favor of circuit-friendly Poseidon.

******************Proof generation time, in seconds (halo2)******************

$$
\begin{array}{|l|c|c|c|c|}
\hline\text{Scheme \textbackslash Variables} & 14 & 16 & 18 & 20 \\\
\hline\text{Hyrax*}       & 64.28 & 71.18 & 139.98 & 261.66 \\\
\hline\text{KZG** (k=17)}     & 88.2 & 102.24 & 115.02 & 127.8 \\\
\hline\text{Ligero}    & 56.76 & 93.04 & 243.10 & - \\\
\hline\text{Brakedown***} & - & - & - & - \\\
\hline\end{array}
$$

*Hyrax costs are estimated by benchmarking the proving time for a 2 x MSMs of size $2^{n/2}$, for a multilinear polynomial in $n$ variables. We adapt the code from [halo2-lib](https://github.com/axiom-crypto/halo2-lib/blob/15bca77fd383a7cb04099483cb6a3524c3615b0d/halo2-ecc/src/bn254/tests/msm.rs#L67) to perform benchmarks over different number of base-scalar pairs.

**KZG costs are estimated by benchmarking a single pairing from [halo2-lib](https://github.com/axiom-crypto/halo2-lib/blob/15bca77fd383a7cb04099483cb6a3524c3615b0d/halo2-ecc/src/bn254/tests/pairing.rs#L49), and multiplying by $n$. The actual computation of multilinear KZG verification involves a multipairing of $n$ pairs, which in general can be optimized to be faster than $n$ individual pairings as reported here.

***Brakedown - we were already unable to run Ligero verifier for large polynomials with the hardware we selected for benchmarking. This is due to over 250+ halo2 advice cells (for $2^{18}$), most of which come from the huge amount of hashing that needs to be performed in the non-interactive setting of linear-code based PC schemes. Brakedown performs at least an order of magnitude more column openings than Ligero, under reasonable choice of parameters for Ligero, and since the verifier needs to hash each column, we conclude that it would be infeasible within the given server setting.

In all estimates, we only focus on the key cost (MSM/pairing) and completely disregard auxiliary costs e.g. to make the scheme non-interactive.

### Objective 4.2: Recursive verifier compute cost

This refers to the outer-verifier costs. As noted above, in the recursive setting, the inner-verifier doesn’t matter much (aside from offline sanity checks if anything).

For all circuits we ran in halo2, we only ever encountered $<1s$  runtimes. We conclude that even when running this in an EVM-verifier, the gas costs will be acceptable and we don’t delve on the details.

### Objective 5: Recursive verifier proof size

The halo2 proof sizes are tightly coupled with the parametrization of the halo2 circuit, namely the (log of the) maximum number of rows, `k`, in the witness generation table. Rather than reporting the proof size for varying `k's` , we instead only present the size for the choice of `k` that results in the fastest prover, as noted in [Objective 4.1: Recursive prover compute cost](https://www.notion.so/Objective-4-1-Recursive-prover-compute-cost-6a64661706fa42bb8965af6a85966f67?pvs=21). The estimates for KZG, Hyrax and Brakedown follow the same methodology.

**************************************Proof size in bytes (halo2)**************************************

$$
\begin{array}{|l|c|c|c|c|}
\hline\text{Scheme \textbackslash Variables} & 14 & 16 & 18 & 20 \\\
\hline\text{Hyrax}       & 9856^* &  34624 & 67712 & 67264^* \\\
\hline\text{KZG (k=17)} & 124096 & 141824 & 159552 & 177280 \\\
\hline\text{Ligero}    &  26048 &  46112 & 85536 & - \\\
\hline\text{Brakedown**} & - & - & - & - \\\
\hline\end{array}
$$

$*$The fastest Hyrax verifier for $n=14$ and $n=20$ uses `k=20`, whereas the middle values for $n$ have the fastest runtimes with `k=19`. This explains the non-doubling (roughly) proof sizes as we increase $n$.

**As for the recursive prover time, we ran out of memory for the largest instance of Ligero, and we expect the same for Brakedown.

## Conclusions

Picking the right PCS is hard. 

While implementing the above schemes, we realized that there are a ton of code optimizations to be done for each, which depend on the specific optimization target (prover/verifier time etc.). The above benchmarks are just the beginning. While many of the opening proofs generation instances could be batched in the case of Lasso, the commitments have to be done to each polynomial separately.

On-chain verification of ZK proofs might be a fun application that has certainly accelerated the progress in the space of SNARKs. Nevertheless, my personal take is that the future users of this technology will be off-chain, e.g. an AI judge (see my application to the startup school) or proof-of-identity. We need to open up to the possibility of not having minuscule proofs; sub-second verification times might be unnecessary; and the real bottleneck becomes the prover.

Linear-based codes offer good tradeoffs in this space: high parallelism thanks to their matrix structure; flexibility from choosing the code distance; plus some schemes like Brakedown also are field-agnostic (no FFTs). I’ll be surprised if we don’t see an order of magnitude improvement on both the theory and implementation side within the next year or two. Till then, there is no clear winner.

### Notes

All our benchmarks were run on the following dedicated instance:

AMD Ryzen™ 9 7950X3D
- CPU: 16 cores / 32 threads @ 4.2 GHz

- Generation: Raphael (Zen 4) with AMD 3D V-Cache™ Technology

- RAM: 128 GB ECC DDR5 RAM

## Future work

- Zeromorph: transformation from any univariate to multilinear scheme. Does applying Zeromorph to known univariate schemes bring better performance?
- Can we introduce any modifications to linear-code based PCS to benefit from “small” coefficients?
- In next steps, we would like to provide benchmarks over BLS12-381. This curve was proposed by the Zcash team as a potential replacement for BN254, whose security is estimated to be [below the initially assumed 128 bits](https://github.com/zcash/zcash/issues/714). There are [some projects](https://eips.ethereum.org/EIPS/eip-2537) underway to bring BLS12-381 curve operations as precompiles on Ethereum. The curve’s future on Ethereum is uncertain, but it has already found its way into other blockchains.
- Upgrade to a newer circuit friendly hash function. Some of the new double-friendly construction attempt to aim for a tradeoff between native and in-circuit efficiency.
- More schemes and more variants!