## Overview


Reference: https://eprint.iacr.org/2019/099.pdf

This document is not intended to be comprehensive. Rather, the process of creating it should help me understand Sonics. The paper describes multiplicative groups but I will use additive groups for notational convenience.

Sonic SNARKS are a construction of zkSNARKs that uses a universal and updatable structured reference string (the public params):
   * Universal: the same reference string supports multiple different functions
   * Updatable: after the string is generated, anyone can contribute their own entropy to update it
   * The SRS is linear with respect to the number of constraints (instead of quadratic as per Groth et al)

## Polynomials

* The polynomials that is defined by the instance of the language (the SNARK solution) is compromised of bivariate monomials: each term has the form a<sub>i j</sub>X<sup>i</sup>Y<sup>j</sup>
* This means that an SRS that contains all of the hidden monomials can be used to commit to a particular instance (if this doesn't make sense, review the SNARKS.md and SNARKS.tex documents)
* Any SRS that only contains monomials is updatable (yet to be shown)
* The polynomial that is determined by the constraints is publicly known (at least to the verifier). We use this to avoid putting constraint-specific terms in the SRS.
* The solution polynomial can be committed to using a univariate scheme: each term has the form a<sub>i</sub>X<sup>i</sup>Y<sup>i</sup>
* A rough outline of the scheme is:
   * the prover commits to the constraint polynomial
   * they generate a random point y in the clear (using the Fiat-Shamir heuristic)
   * they can now commit to a polynomial with terms of the form t<sub>i j</sub>X<sup>i</sup>y<sup>j</sup>
* Either prover or verifier can compute the constraint polynomial so we either:
   * make the prover do it and then prove they did it correctly
   * a "helper" aggregates lots of proofs and then proves all polynomial evaluations simultaneously (the circuit-dependent cost is amortised over many proofs)

## Related Work
* Can't be bothered summarizing this

## Updatable reference strings
* Consider a setup string `srs` and a proof of its correctness `ρ`
* The update operation takes an SRS and a series of the proofs so far to produce a new `srs'` and `ρ'`
* Naturally, there is a verify algorithm that confirms the proof matches the SRS.
 
## Bilinear Groups
A bilinear group is comprised of:
* a prime p
* three groups G<sub>1</sub>, G<sub>2</sub> and G<sub>T</sub>
* generators for the first two groups **g** and **h** respectively
* a pairing function e that maps pairs of elements from the first two groups to the third one and is bilinear:
   * e(a⋅**g**, b⋅**h**) = ab⋅**e(g,h)**
   * **e(g,h)** is a generator for G<sub>T</sub>

## Structured Reference String
The SRS is composed of
* x<sup>i</sup>⋅**g** for all integer i between -d and d (the polynomial degree)
* αx<sup>i</sup>⋅**g** for all integer i between -d and d **except 0**
* x<sup>i</sup>⋅**h** for all integer i between -d and d 
* αx<sup>i</sup>⋅**h** for all integer i between -d and d

Note: since there is no α⋅**g** term we can always force someone to prove a polynomial has no constant term.

## Constraints

Note: After reviewing this section it's still not clear to me what they're trying to do. This system is based on another paper (https://eprint.iacr.org/2016/263.pdf) so I will review that in zkCircuits.md


The constraint system can be represented by the vector equation **a**◦**b** = **c** and Q linear constraints of the form
* **a**⋅**u<sub>q</sub>** + **b**⋅**v<sub>q</sub>** + **c**⋅**w<sub>q</sub>** = k<sub>q</sub> where the subscripted vectors are public constants for the q-th constraint.

For example, we can represent x<sup>2</sup> + y<sup>2</sup> = z with the system:
* **a** = (x,y), **b** = (x,y), **c** = (x<sup>2</sup>, y<sup>2</sup>)  ( the quadratic constraints )
* **u<sub>1</sub>** = (1, 0), **v<sub>1</sub>** = (-1, 0), **w<sub>1</sub>** = (0, 0), k<sub>1</sub> = 0 (ie. _x + x - 0 = 0_)
* **u<sub>2</sub>** = (0, 1), **v<sub>2</sub>** = (0, -1), **w<sub>2</sub>** = (0, 0), k<sub>2</sub> = 0  (ie. _y + y - 0 
* **u<sub>3</sub>** = (0, 0), **v<sub>3</sub>** = (0, 0), **w<sub>3</sub>** = (1, 1), k<sub>3</sub> = z  (ie. _x<sup>2</sup> + y<sup>2</sup> = z)

As far as I can tell, the first two equations are wasted because we are using hardcoded vectors representing various linear combinations. I suspect the full set of vectors is designed so every linear combination is possible.

The multiplication vector equation represents n different constraints. We can use each as a coefficient of a polynomial in Y. When all equations are satisfied, this polynomial is 0:

sum<sub>i</sub>\[ (a<sub>i</sub>b<sub>i</sub> - c<sub>i</sub>)Y<sup>i</sup> ] = 0

We will also define the redundant polynomial equation using negative exponents of Y:

sum<sub>i</sub>\[ (a<sub>i</sub>b<sub>i</sub> - c<sub>i</sub>)Y<sup>-i</sup> ] = 0

We can also encode the Q linear constraints as a polynomial in Y, scaling the powers of Y<sup>n</sup> so the don't interfere with each other:

 sum<sub>q</sub>\[ (**a**⋅**u<sub>q</sub>** + **b**⋅**v<sub>q</sub>** + **c**⋅**w<sub>q</sub>** - k<sub>q</sub>)Y<sup>q+n</sup> ] = 0

Note these equations satisfy all constraints simultaneously. 

We can group all the multiplicative terms corresponding to the i-th variable into its own polynomial:
* u<sub>i</sub>(Y) = sum<sub>q</sub>\[  Y<sup>q+n</sup> u<sub>q,i</sub> ]
* v<sub>i</sub>(Y) = sum<sub>q</sub>\[  Y<sup>q+n</sup> v<sub>q,i</sub> ]
* w<sub>i</sub>(Y) = -Y<sup>i</sup> - Y<sup>-i</sup> + sum<sub>q</sub>\[  Y<sup>q+n</sup> w<sub>q,i</sub> ]
* k<sub>i</sub>(Y) = sum<sub>q</sub>\[  Y<sup>q+n</sup> k<sub>q</sub> ]

Note that the additional terms in the w polynomials are designed cancel out the terms in the linear constraints.

We can now write our constraints as the equation:

**a**⋅**u**(Y) + **b**⋅**v**(Y) + **c**⋅**w**(Y) + sum<sub>i</sub>\[  a<sub>i</sub>b<sub>i</sub>(Y<sup>i</sup> + Y<sup>-i</sup>)  ] - k(Y) = 0

So given a choice of (**a**, **b**, **c**, k(Y)), this equation holds if and (probably only if) the choice satisfies all constraints.

We now want to embed the LHS of this equation into the constant term of a polynomial t(X,Y):
* define polynomial r such that r(X, Y) = r(XY, 1) - the terms include the same power of both X and Y.
* r(X,Y) = sum<sub>i=1...n</sub> \[  a<sub>i</sub>X<sup>i</sup>Y<sup>i</sup> + b<sub>i</sub>X<sup>-i</sup>Y<sup>-i</sup> + c<sub>i</sub>X<sup>-i-n</sup>Y<sup>-i-n</sup>  ]
* s(X,Y) = sum<sub>i=1...n</sub> \[  u<sub>i</sub>(Y)X<sup>-i</sup> +  v<sub>i</sub>(Y)X<sup>i</sup> +  w<sub>i</sub>(Y)X<sup>i+n</sup>  ]
* r'(X,Y) = r(X,Y) + s(X,Y)
* t(X,Y) = r(X,1)r'(X,Y) - k(Y)

Importantly, the coefficient of X<sup>0</sup> in t is the LHS of our final constraint equation. The Sonic protocol demonstrates that the constant term of t(X,Y) is zero, which proves the original constraint system is satisfied.


## The protocol
* The prover constructs r(X,Y) using their hidden witness
* They commit to r(X, 1)  \[note: this is a polynomial]
* The verifier sends a random challenge `y`
* The prover commits to t(X, y) and the scheme ensures that this polynomial has no constant term.
* The verifier sends another challenge `z`
