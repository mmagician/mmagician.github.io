---
title: "Arkworks"
date: 2022-07-27T11:05:21+02:00
draft: false
---

# Arkworks

[arkworks on GitHub](https://github.com/arkworks-rs)

## Small start

My first contribution to arkworks was improving the [documentation of finite fields](https://github.com/arkworks-rs/algebra/pull/334).
Many public structs and traits were at that time without any description and it was difficult, especially for newcomers to the arkworks ecosystem like myself, to grasp the complexity, let alone use the APIs. One had to resort to reading the source code!

Later I added doctests (a code example included in the documentation of that item, which actually executes when you `cargo test`) and fixed some CI jobs, so that the repo appeared all nice and green.

## Bugs

As you're writing documentation, and especially doctests, you are likely to come upon edge cases. I did just that, and fixed a couple of bugs t, like [#358](https://github.com/arkworks-rs/algebra/pull/358) or [#366](https://github.com/arkworks-rs/algebra/pull/366). As with most well-hidden bugs, these mostly required in-depth understanding of the algorithm involved and all the possible scenarios. For example, the first one mentioned works fine in case where the second coefficient of the quadratic extension field was `0`, and the first was a square root in the base field, but would return an incorrect result if the first coeff was not a square root in base, but only in the extension field. I've also added an accompanying regression test [here](https://github.com/arkworks-rs/curves/pull/89).

## Feature fun

Later on, as I've gained a familiarity with the arkworks ecosystem, I got involved in adding new features as well as improving the API to be more user friendly and to make better use of the newest and greatest Rust features.

Some examples of refactorings I've done include: [SquareRootField & Field unification](https://github.com/arkworks-rs/algebra/pull/422), [Refactoring the MSM interface](https://github.com/arkworks-rs/algebra/pull/425) or [introducing an explicit cofactor clearing method](https://github.com/arkworks-rs/algebra/pull/420) (for which I've also added a [~5x improvement](https://github.com/arkworks-rs/curves/pull/103/) to one of the curves).

The biggest "new" feature I contributed was hash-to-field in two parts: [hash-to-curve](https://github.com/arkworks-rs/algebra/pull/405) & [map-to-field](https://github.com/arkworks-rs/algebra/pull/430). Kudos to my colleagues at the W3F for the early work and theoretical support on those, especially regarding isogenies.
