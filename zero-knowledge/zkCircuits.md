## Overview

Reference: https://eprint.iacr.org/2016/263.pdf

I'm trying to understand Sonic Snarks (Reference: https://eprint.iacr.org/2019/099.pdf) but the constraint system is still unclear. It is based off this paper so I will review it first.

This document is not intended to be comprehensive. Rather, the process of creating it should help me understand the constraint system.

## Preliminaries

### Notation

* **g** is a vector of n values in prime order group G
* **f** is a vector of n values in the group of integers mod p
* **g<sup>f</sup>** is the product of all (g<sup>f</sup>)<sub>i</sub> values

### Assumptions

* The standard Discrete Logarithm assumption: an adversary cannot find an `a` such that <code>g<sup>a</sup> = h</code> for a given `h`
* The Discrete Logarithm relation assumption: given a set of `g` values in G, an adversary cannot find `a` values such that the product of (g<sup>a</sup>)<sub>i</sub> is 1

### Commitments
* a prover commits to a secret value in such a way that they can later reveal it
* a perfectly hiding commitment does not reveal the committed value
* a binding commitment scheme can only be opened (revealed) to one value

#### Pedersen commitment
* c = g<sup>r</sup>h<sup>m</sup> where
   * c is the commitment 
   * r is a random value 
   * m is a message
   * g and h are public group elements
* to open a commitment, simply reveal m and r
* we will use a variant of this scheme that allows us to commit to multiple values at once:
   * c = g<sup>r</sup> * product\[ g<sub>i</sub><sup>m_i</sup> ]
   * here g<sup>r</sup> is the random mask
* Importantly, this is a homomorphic commitment scheme:
   * Com(m<sub>1</sub>,m<sub>2</sub>,m<sub>3</sub>,r<sub>1</sub>) * Com(m<sub>4</sub>,m<sub>5</sub>,m<sub>6</sub>,r<sub>2</sub>) = Com(m<sub>1</sub>+m<sub>4</sub>,m<sub>2</sub>+m<sub>5</sub>, m<sub>3</sub>+m<sub>6</sub>, r<sub>1</sub>+r<sub>2</sub>)
   
### Zero Knowledge argument of knowledge
* complete: a verifier will accept a valid proof. In such a case, the prover must have knowledge of a witness
* an argument is public coin if the verifier chooses challenge messages at random (independent of the prover's messages). In such a case, the Fiat-Shamir heuristic can be used to make the proofs non-interactive. 

## Commitments to Polynomials
* we want to be able to commit to a Laurent polynomial t(X) (X can have negative powers) and then later reveal the evaluation at a particular point x. 
* a straightforward solution is to commit to the coefficients of t(X) individually. We can then:
    * provide a solution to t(z)
    * the verifier can evaluate E(t(z)) using the homomorphic properties and compare
* this required `d` elements to be sent - one per coefficient.
* here we present a mechanism to reduce that to sqrt(d)

### The basic idea

Consider a standard (non-laurent) polynomial with d+1=mn. We can arrange the coefficients in an m x n matrix M. If we multiply the matrix by a column vector of the powers of X, we get m polynomials of degree n-1. If we scale them each by X<sup>in</sup> ie. pre-multiplying by the row vector (1 X<sup>n</sup> ... X<sup>(m-1)n</sup>) we can recover the original polynomial t(X).

The idea here is that we can make m commitments to the rows of the matrix. We can then use the homomorphic property to compute the commitment to the full coefficient vector. The prover can open this commitment (m values) and then anyone can compute t(x).

To avoid leaking information about the coefficients of t(X) we add some blinding values u<sub>i</sub> and then add another row to undo the effect of the changes.

### An example

t(X) = t<sub>0,0</sub>X<sup>0</sup> + t<sub>0,1</sub>X<sup>1</sup> + t<sub>0,2</sub>X<sup>2</sup> + 

  X<sup>3</sup>(t<sub>1,0</sub>X<sup>0</sup> + t<sub>1,1</sub>X<sup>1</sup> + t<sub>1,2</sub>X<sup>2</sup>) + 
  
  X<sup>6</sup>(t<sub>2,0</sub>X<sup>0</sup> + t<sub>2,1</sub>X<sup>1</sup> + t<sub>2,2</sub>X<sup>2</sup>)
  
We can rewrite this as:

t(X) = t<sub>0,0</sub>X<sup>0</sup> + (t<sub>0,1</sub> - u<sub>1</sub>)X<sup>1</sup> + (t<sub>0,2</sub> - u<sub>2</sub>)X<sup>2</sup> + 

X<sup>3</sup>(t<sub>1,0</sub>X<sup>0</sup> + t<sub>1,1</sub>X<sup>1</sup> + t<sub>1,2</sub>X<sup>2</sup>) + 

X<sup>6</sup>(t<sub>2,0</sub>X<sup>0</sup> + t<sub>2,1</sub>X<sup>1</sup> + t<sub>2,2</sub>X<sup>2</sup>) + 

X(u<sub>1</sub>X<sup>0</sup> + u<sub>2</sub>X<sup>1</sup> + 0 X<sup>2</sup>)

Notice that we have four polynomials of degree 2. We can make a (multi-value) commitment to each polynomial individually:
* T<sub>0</sub> = g<sup>r_1</sup>g<sub>1</sub><sup>t_0,0</sup>g<sub>2</sub><sup>t_0,1 - u_1</sup>g<sub>3</sub><sup>t_0,2 - u_2</sup>
* T<sub>1</sub> = g<sup>r_2</sup>g<sub>1</sub><sup>t_1,0</sup>g<sub>2</sub><sup>t_1,1</sup>g<sub>3</sub><sup>t_1,2</sup>
* T<sub>2</sub> = g<sup>r_3</sup>g<sub>1</sub><sup>t_2,0</sup>g<sub>2</sub><sup>t_2,1</sup>g<sub>3</sub><sup>t_2,2</sup>
* T<sub>3</sub> = g<sup>r_4</sup>g<sub>1</sub><sup>u_1</sup>g<sub>2</sub><sup>u_2</sup>g<sub>3</sub><sup>0</sup>

Once x is chosen, we can reveal:
* t<sub>0,0</sub> + x<sup>3</sup>t<sub>1,0</sub> + x<sup>6</sup>t<sub>2,0</sub> + xu<sub>1</sub>
* t<sub>0,1</sub> - u<sub>1</sub> + x<sup>3</sup>t<sub>1,1</sub> + x<sup>6</sup>t<sub>2,1</sub> + xu<sub>2</sub>
* t<sub>0,1</sub> - u<sub>2</sub> + x<sup>3</sup>t<sub>1,2</sub> + x<sup>6</sup>t<sub>2,2</sub>

Note that each of these is perfectly hidden individually, but they are correlated so that the hidings cancel out.

The verifier can scale these by powers of x appropriately to recover t(x).  


### Laurent polynomials

Let t(X) = sum\[ t<sub>i</sub>X<sup>i</sup> ] where i can be negative (ranges from -d<sub>1</sub> to d<sub>2</sub>) and t<sub>0</sub> = 0 (no constant term).

We can accommodate these additions in the same scheme by:
* grouping the terms into n-term polynomials as before, scaled by the appropriate powers of X
* note that we start with negative powers of X
* we scale the positive-power terms with X<sup>1</sup>, X<sup>n+1</sup>, ... (instead of starting with X<sup>0</sup>) to account for the fact that there is no constant term
* scale the last U polynomial by X<sup>2</sup> (instead of X) to compensate

### Evaluation protocol

#### Commitment
* we want to commit to a Laurent polynomial t(X) with powers of X ranging from -m<sub>1</sub>n to m<sub>2</sub>n and no constant term (ie. can be grouped into sub-polynomials of degree n-1).
* pick blinding values u<sub>1</sub> to u<sub>n-1</sub> and reframe the polynomials with the blinding factors (adding another sub-polynomial)
* choose a random τ value per sub-polynomial and commit each one individually (including the new **u** one)

#### Polynomial evaluation
* Note that each term in the full polynomial t(X) can be thought of as the product of:
   * a coefficient t<sub>i,j</sub>
   * a power of x in the sub-polynomial (up to n-1)
   * a shift of some power of x to move the sub-polynomial to the right place in the full polynomial
* we can group these terms by their index in the sub-polynomial (n-1 summations).
* once x is chosen, we can substitute it for all the shifts but NOT the X value in the sub-polynomials, creating a single polynomial of degree n-1.
* note that each coefficient is masked with one or more u values
* we reveal these coefficients, but the mask ensures that the original t coefficients are still hidden
* we also treat the τ values as the coefficients of a polynomial in X and reveal the result of this polynomial evaluated at x (ie. sum\[ τ<sub>i</sub>x<sup>i</sup> where i ranges from from -m<sub>1</sub>n to m<sub>2</sub>n except i=0 ])

#### Verification
* the verifier has a series of commitments to the sub-polynomials.
* they can scale each commitment by powers of the chosen point x so these become commitments to the individual terms in t(x) \[still grouped in the sub-polynomials]. Note that this process inadvertently scales the random τ values as well.
* they can now multiply the commitments together, creating a single commitment where:
   * the random value is the sum of the individual random values scaled by powers of x
   * the i-th component of the commitment is the scaled sum of all the i-th coefficients across all sub-polyomials
* this commitment should be equivalent to a commitment of the values revealed in the polynomial evaluation step
* if they match, the verifier believes the polynomial evaluation step was done correctly (it matches the original commitment)
* the verifier has coefficients of an n-1 degree polynomial P(X) that evaluates to the larger polynomial t(x) at x.
* they calculate P(x) directly to compute t(x)
