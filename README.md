# Keraunos

Keraunos is the distributed **Pollard kangaroo** search of the [BitBOINC](https://bitboinc.athena.org.tr/) project. It looks for the private key of a Bitcoin puzzle address whose **public key is already known** and whose private key is known to lie in a limited interval.

> **This repository contains documentation only. It does not contain source code, work-unit parameters, searched ranges or server code.** See [What is not published, and why](#what-is-not-published-and-why).

## The problem

For a puzzle the key lies in an interval `[a, b]` and the public point `Q = k·G` on secp256k1 is known. Finding `k` is a discrete-logarithm problem in a short interval. The best known generic method is Pollard's kangaroo (lambda) algorithm, which needs on the order of the **square root of the interval width** group operations, instead of the interval width itself that brute force needs.

## The method

- **Tame and wild kangaroos.** Tame kangaroos start from points whose discrete logarithm is known inside the interval. Wild kangaroos start from `Q` plus a known offset. Each kangaroo repeatedly jumps by a pseudo-random step chosen from a small table, selected by a hash of its current point, so two kangaroos that ever land on the same point follow the same path from then on.
- **Distinguished points (DPs).** Storing every point would be impossible. A point is "distinguished" when its encoding has a fixed number of leading zero bits. Only DPs are reported, together with the distance the kangaroo has travelled.
- **Collision.** When a tame and a wild kangaroo report the same DP, the difference of their travelled distances gives `k`.
- **Many walkers, one search.** Every task runs a large herd of kangaroos. All CPU and GPU versions use the same jump rule and the same DP rule, so a CPU walk and a GPU walk can collide with each other. The DP threshold is a tuning knob that trades network traffic against the work lost to unfinished walks.

## How it runs on BOINC

- Volunteers' computers run the kangaroo walks (CPU on many platforms, GPU through CUDA, OpenCL and Metal).
- Found DPs are sent to the project server in small pieces as BOINC *trickle* messages, while the task is still running. A server-side daemon stores them and looks for tame/wild collisions.
- GPU herds are kept on disk between tasks, so an interrupted task does not throw the herd away.
- One puzzle is worked on at a time. Credit is a flat amount per task.

## Correctness and safety

- **Self-tests on the volunteer's machine.** Before a task starts walking, the program runs a known-answer test of its own arithmetic. Some GPU drivers and compilers miscompute 256-bit arithmetic; on such a device the task refuses to run, so that no wrong points are ever sent.
- **Server-side verification.** A reported collision is turned into a candidate key `k`; the server checks that `k·G` equals the target public key before it counts the puzzle as solved. Wrong or fake reports therefore cannot mark a puzzle as solved.

## What happens if a key is found

- The candidate key is **encrypted immediately with an RSA-4096 (OAEP) public key**. Only the **ciphertext** is stored (database, log and an extra backup file); the plaintext key never touches the server's disk or logs.
- The matching **private key is kept offline** by the project owner and has never been on the server. Nobody who gets access to the server, the database or the logs can read a found key.
- Notifications about a solve never contain the key.
- The key is only used for puzzles that were created to be solved. The project does not search for the keys of arbitrary addresses.

## What volunteers' computers do and do not do

They compute walks and send DPs. They do not mine, do not touch any wallet or file of the user, and the project stores no personal data about the search itself. See the project's privacy page.

## What is not published, and why

The source code of the optimized implementation, the tuning parameters and the search state are deliberately **not published**. A fast kangaroo implementation is a competitive advantage in a race for a prize that goes to the first finder, and per-key data (DPs, ranges) would let others skip or copy the work. The correctness and security design above is published so that it can be reviewed without the solver.

## License

The text of this repository is released under the GPL-3.0 (see `LICENSE`).
