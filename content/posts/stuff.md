---
title: "From one {point,poly} to multi {point,poly}"
date: 2022-08-01T11:03:00+02:00
draft: false
math: true
---

This is my first entry here, so apart from your understanding, please give me feedback so I can improve in the future.

This is not an introduction to PCS. Instead, I will use some statements from vanilla KZG and build upon these for multiple points and multiple polynomials. If you are new to PCS, I recommend the intro by [Dankrad Feist](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html) or the [original KZG paper](https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf).

# Polynomial committment schemes

The primary focus of this article is to explore various flavours of KZG polynomial committment scheme:
- single polynomial, single opening (vanilla)
- single polynomial, multiple openings
- multiple polynomials, single opening
- multiple polynomials, multiple openings

and show how the committment changes, how are the proofs constructed, and what the verifier needs to do to get convinced of the proof's validity.
## Vanilla

Here I just re-state the results from KZG and explain some notation:

- secret $s$ from SRS: $ \{ G, sG, s^2G, ..., s^{n-1}G \} $
- polynomial $p(X)$
- commitment, $C = [p(s)]_1$
- claim $p$ at $z$: $y = p(z)$
- quotient polynomial $h(x) = \frac{p(x) - y}{x-z}$
- proof, $\pi = [h(s)]$

One thing worth noting is that the quotient polynomial needs to be computed separately for each point $z$ (if we evaluate at multiple points).

Verifier's job is to perform a pairing check:

$$ e(C - [y]_1, G_2) \stackrel{?}{=} e(\pi, [s-z]_2)$$

## Single polynomial, multiple openings

Perhaps the easiest extension of the above scheme is verifying the polynomial $p$ at multiple points $z_1, ..., z_t$. Let's consider the case $t=2$, where we have just two points.
Of course the prover could provide two proofs $\pi_1$ & $\pi_2$ to prove that $p(z_1) = y_1$ & $p(z_2) = y_2$, but we can be more efficient than that.

Recall that for a single opening at $z$, we construct a polynomial $p(x) - p(z)$ which clearly evaluates to $0$ if $x=z$, i.e. it has a factor of $x-z$.
We would like to extend this to two points now, so we require that some polynomial $p(x) - r(x)$ evaluates to zero at both $z_1$ & $z_2$. The way to find what $r(x)$ needs to be is rewriting that condition:

$p(z_1) = r(z_1)$, $p(z_2) = r(z_2) $

$ y_1 = r(z_1) $, $y_2 = r(z_2) $

and noticing that $r(x)$ needs to pass through two points $(z_1, y_1)$ and $(z_2, y_2)$ - it's a line! We can use Lagrange interpolation to compute $r(x)$.

So we have that $(x-z_1)\cdot(x-z_2)$ is a factor of $p(x) - r(x)$. For the case of multiple openings, we define $h(x)$ slightly differently:

$$h(x) = \frac{p(x) - r(x)}{Z_s(x)}$$

where $Z_s(x) = (x-z_1)\cdot(x-z_2)$

Prover's definitions of committment $C$ and proof $\pi$ don't change from the basic version.
The verifier's check is only slightly different to include the fact that we no more have a single $p(z) = y$, but rather $r(x)$, and the different $G_2$ element on the RHS of the pairing equation above:

$$ e(C - [r(s)]_1, G_2) \stackrel{?}{=} e(\pi, [Z_s(s)]_2)$$

The argument here can be easily extended to more than two points: $r(x)$ then is a quadratic, cubic, ..., degree-(t-1)-polynomial instead of a line (degree-1-polynomial).

## Multiple polynomials, one opening

I'm not sure whether this has any practical applications, but it serves well as an intermediate step on our way to opening multiple polynomials at multiple points.

Instead of dealing with a single polynomial $p(x)$, the prover now wants to commit to multiple polynomials (let's assume they have the same degree) and later prove their openings at a single point $z$.

The committment phase changes, as now for each polynomial $p_i(x), i  \in {1, ..., k}$, we generate $C_i = [p_i(s)]$.

Reall that for vanilla KZG, we constructed $p(x) - p(z)$ s.t. when divided by its roots $(x - z)$, returns a quotient polynomial $h(x)$ with no remainder, or in other words:
$$ p(x) - p(z) = 0 \mod (x-z)$$

and that it only happens at a maximum of $d = |degree(p)|$ points, or in other words with probability $\frac{d}{𝔽}$. This should give the verifier confidence that the proof $[h(s)]_12$ isn't just a random group element, but rather that the prover knows the original polynomial $p$ and can compute its evaluations at challenge points $z$.

We define $h(x)$ as:
$$h(x) = \sum_{i=1}^{k}\gamma^{i-1}\frac{p_i(x) - y}{x-z}$$

To see why, let's digress slightly and define an arbitrary $Z(x) = (x-z)$ and:
$$ G(x) = \sum_{i}^{k} \gamma^{i-1} \cdot F_i(x)$$

Notice that if even all $F_i$ were divisible by $Z(x)$ except, say, $F_j$, then the probability that $Z(x) | G(x)$ is $\frac{k}{𝔽}$, because:
$$ F_j \mod Z(x) = R(x) \neq 0$$ 
$$ G(x) = \sum_{i \neq j} \gamma^{i-1} \cdot F_i(x) + \gamma^{j-1}\cdot R(x) \mod Z(x)$$

And for some specific $\Chi \in 𝔽$ s.t. $Z(\Chi) = 0$ but $R(\Chi) \neq 0$, if we wish $G(\Chi) = 0$, then with a slight abuse of notation we have:
$$ 0 = \sum_{i \neq j} \gamma^{i-1} \cdot F_i + \gamma^{j-1}\cdot R$$

which is just a polynomial in $\gamma$ of degree $k$, which has at most $k$ solutions.

The point of all that being: the verifier can compute $Z(x) = (x-z)$ by themselves and so if at least one of the committed polynomials is not divisible by (x-z), then the prover will most likely be caught.

Let's wrap up this section with the pairing check:

$$
\begin{aligned} e(C_i - [y]_1, G_2) \cdot ... \cdot e(C_k - [y]_1, G_2) \stackrel{?}{=} e(\pi, [s-z]_2) \end{aligned}\label{4}\tag{4}
$$

## Multiple polynomials, multiple openings

(Note that multiple variants exist with different tradeoffs, here I present the first scheme from [this paper](https://eprint.iacr.org/2020/081.pdf)).

As with the preceding scheme, the prover commits to each polynomial $p_i(x)$ separately: $C_1, ..., C_k$.

We will need to define a set of all evaluation points $T$ as $T = \lbrace z_1, ..., z_t \rbrace$. 

The quotient polynomial $h(x)$ is a straightforward combination of two above two schemes: we take advantage of both the sum over $p_i(x)$ as well as the generalised $r_i(x)$ and $Z_{s_i}(x)$, except that I've added an extra subscript to indicate that each polynomial might have a different set of evaluation points. This means that each polynomial $p_i$ gets evaluated not necessarily on the entire domain $T$, but its subset $S_i$, and that these subsets can overlap.

(for example, $p_1(x)$ is opened at $S_1 = \lbrace z_1, z_2 \rbrace$, and $p_2(x)$ is opened at $S_2 = \lbrace z_1, z_3 \rbrace$).

$$
\begin{aligned}
h(x) = \sum_{i=1}^{k}\gamma^{i-1}\frac{p_i(x) - r_i(x)}{Z_{s_i}(x)} \end{aligned}\label{5}\tag{5}
$$

Unfortunately, having distinct openings for each $p_i$ complicates the pairing check, since we cannot just plug in $Z_{s}$ into the RHS of $\eqref{4}$, because now $Z_{s_i}$ are distinct!

However, imagine for a minute that we allow ourselves to compute a product of pairings on the RHS (instead of a single pairing), where we define each $\h_i$ as the fraction from the sum of $\eqref{5}$, and $pi_i$ as its commitment.

$$
\begin{aligned}
h_i = \frac{p_i(x) - r_i(x)}{Z_{s_i}(x)} \end{aligned}\label{}\tag{}
$$

Then we have 
$$
\begin{aligned}
e(C_1 - [r_1(s)]_1, G_2) \stackrel{?}{=} e(\pi_1, [Z_1(s)]_2) \newline 
e(C_2 - [r_2(s)]_1, G_2) \stackrel{?}{=} e(\pi_2, [Z_2(s)]_2) \newline
... \newline
e(C_k - [r_k(s)]_1, G_2) \stackrel{?}{=} e(\pi_k, [Z_k(s)]_2) \newline 
\end{aligned}\label{4}\tag{4}
$$

Which is checking the polynomial equality:
$$h_i(x)\cdot Z_{s_i} = p_i(x) - r_i(x)$$
We can use a trick to multiply both sides by $Z_T(x) = (x-z_1)\cdot...\cdot(x-z_t)$:
$$h_i(x)\cdot Z_{s_i} \cdot Z_T(x) = (p_i(x) - r_i(x))\cdot Z_T(x)$$

and then divide by $Z_{s_i}$:
$$h_i(x)\cdot Z_T(x) = (p_i(x) - r_i(x))\cdot Z_{T/S_i}(x)$$

and $Z_i = Z_{T/S_i}(x)$ is defined as $\prod_{i \notin S_i}{(x-z_i)}$

Same thing in pairing notation:

$$
\begin{aligned}
e(C_k - [r_k(s)]_1, [Z_i(s)]_2 ) = e(\pi_k, [Z_T(s)]_2) \newline 
\end{aligned}
$$

But we can combine the RHS for all $i \in \lbrace 1, ..., k\rbrace$ as:

$$
e(C_1 - [r_1(s)]_1, [Z_1(s)]_2) \cdot ... \cdot e(C_k - [r_k(s)]_1, [Z_k(s)]_2 ) = e(\pi, [Z_T(s)]_2)
$$
