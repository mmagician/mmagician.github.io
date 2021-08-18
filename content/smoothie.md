---
title: "ZK Sudoku Game"
date: 2021-08-11T19:14:50+02:00
draft: false
---
# Smoothie - Sudoku Made Out Of The Hard mathematIcal Elements

Zero Knowledge Proofs are still mysterious and hard to explain to non-cryptographers. I am building a user friendly platform to demonstrate the principle behind ZKPs as a game between a prover & a verifier.

To this end, I am using a well-known game of sudoku: users will solve the sudoku game in their browser and generate a proof of correctness. The verifier will be a smart contract that can verify the proofs and issue acknowledgements for valid solutions.

## Project details

The game replies on solving publicly available sudoku puzzles. An _arbitrarily_ large number of games can be made available to the players, since the number of possible sudoku games is > 6e22.

The concept revolves around proving that you know a solution to a particular puzzle, without actually disclosing this solution. Most importantly here, verification of a solution happens in a non-interactive fashion, i.e. there is only one round of communication between the prover and the verifier, namely sharing the proof.

A smart contract can hold the number of different puzzles (perhaps as hashes), together with a list of correct submissions for each puzzle.

Since the state of the smart contract is public, anyone can inspect how many other players have already solved a particular puzzle, as well as verify each individual submission - yet without learnign the solution for themselves.

All code will be open sourced. Wherever it makes sense (mostly M1-M3), we will fork from various Dusk repos (mostly [plonk gadgets](https://github.com/dusk-network/plonk_gadgets/)). Upstream contributions will be made when appropriate.

# Roadmap

## Part I - ZK circuit design
### Milestone 1 - Intermediary gadgets (done :white_check_mark:)
- set membership
- set non-membership
- set uniqueness (no duplicate elements in vector)
- working example
- unit tests

### Milestone 2 - Sudoku-tailored gadgets (in progress :construction:)
- each entry in a 9-element vector between 1-9 and unique
- working example
- unit tests

### Milestone 3 - Full sudoku gadget
- combine the above gadgets
- working example with a prover & verifier for a sudoku puzzle
- unit tests

## Part II - UI, Browser Integration & Deployment
### Milestone 4 - Browser-friendly prover (in progress :construction:)
- compile the prover code to WASM
- expose the prover interface to WASM, so that it can be called from JS
- optimise the Rust program for size and performance
- provide build & deploy (UI + prover WASM) instructions

### Milestone 5 - UI + CLI verifier (in progress :construction:)
- build out a UI in React for interactive sudoku solving
- ability to generate proofs (call the WASM prover)
- at this stage, proofs can be verified by a CLI tool

### Milestone 6 - On-chain Verifier
- investigate the options for including the verification logic on-chain: Ethereum smart contract (akin to Tornado.cash), Polkadot substrate module (like [anon](https://github.com/webb-tools/anon)) or Rusk
- implement the verifier and deploy to a test/mainnet

### Milestone 7 - Touch ups & optimisation
- send proof of solvability together with the puzzle
- investigate optimisation of the gadgets: can the circuit size be decreased?
- user testing
