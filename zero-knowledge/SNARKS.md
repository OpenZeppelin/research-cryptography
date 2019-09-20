
### Overview

This document tracks working thoughts. It is not intended to be comprehensive, rather the process of creating it should help me understand SNARKs.

### RSA

Recall the RSA algorithm:
  * _p_ and _q_ are primes
  * _N = pq_
  * _d_ is a random number (the private key)
  * _e = d<sup>e</sup> mod N_

Encryption of message _m_:

_E(m) = m<sup>e</sup> mod N_

Decryption:

_D(m) = E(m)<sup>d</sup> mod N_ = m<sup>ed</sup> mod N = _m_

#### Multiplicatively homomorphic

Note that RSA is multiplicatively homomorphic:

_E(x) • E(y) = x<sup>d</sup> • y<sup>d</sup> = (x•y)<sup>d</sup> = E(x•y)_

### Zero Knowledge

Consider a function representing a possible Ethereum state transition _f(σ<sub>1</sub>, σ<sub>2</sub>, s, r, v, p<sub>s</sub>, p<sub>r</sub>, v)_ that returns _1_ if:
* _σ<sub>1</sub>_ and _σ<sub>1</sub>_ are the merkle roots of the pre and post account Merkle trees
* _s_ and _r_ are the sender and receiver accounts
* _p<sub>s</sub>_ and _p<sub>v</sub>_ are Merkle proofs testifying that _f_ represents a valid transition:
   * _s_ has a balance of at least _v_ in _σ<sub>1</sub>_
   * _σ<sub>2</sub>_ is the result of transferring _v_ from _s_ to _r_

Note that this is easy to evaluate if all of the inputs are known. On the other hand, if only the Merkle roots are known, then the remaining parameters form a _witness string_: private knowledge that would make _f_ return _1_. If someone can prove that they have a witness string, the verifier knows that _σ<sub>1</sub>_ to _σ<sub>1</sub>_ is a valid transition without knowing the value or the participants in the transaction. Note: this is how [the blog I'm reading](https://blog.ethereum.org/2016/12/05/zksnarks-in-a-nutshell/) describes the process but presumably _f_ should also accept a signature that proves _s_ agreed to the transition.

### Complexity Theory

Define a _problem_ as a function that outputs only _0_ or _1_. If a function can be executed in at most _n<sup>k</sup>_ steps on inputs of size _n_, we call this a polynomial-time program.

**P** is the class of problems **L** that have polynomial-time programs.

**NP** is the class of problems **L** that have a polynomial-time program _V_ that can be used to verify something given a polynomial-sized witness for that fact. **NP** = { L(X) = 1 iff ∃ w such that V(x,w) = 1 }

#### P = NP?

Note that **P** is a subset of **NP** (you could think of **P** as the subset of **NP** that has zero-length witnesses). Most people believe it is a strict subset (there are problems in **NP** that are not in **P**) but this has not been proven. Nevertheless, cryptography relies on this assumption.

For example, consider the problem of finding the private key associated with an Ethereum address:
* This is clearly in **NP** (the private key itself acts as witness - if you have it, you can verify that it is accurate with a polynomial-time program)
* If **P = NP**, there must be a polynomial-time program that can find the private key without any witness (ie. without knowing the private key beforehand)

#### Satisfiability (SAT)

Define a boolean formula:
* any variable _x<sub>1</sub>_, _x<sub>2</sub>_, _x<sub>3</sub>_... is a boolean formula
* if _f_ is a boolean formula, then _¬ f_ is a boolean formula
* if _f_ and _g_ are boolean formulas, (f ∧ g) and (f ∨ g) are boolean formulas.

A boolean formula is satisfiable if there is a way to assign truth values to the variables such that the formula evaluates to `true`

##### SAT is NP Complete

This means that any problem in NP can be transformed to an equivalent satisfiability problem.

### Quadratic Span Programs

These are **NP**-complete programs that are particularly well suited for zKSNARKS:
* define a field _F_ with inputs of length _n_
* define a set of basis polynomials _v<sub>0</sub> ,..., v<sub>m</sub> , w<sub>0</sub> ,..., w<sub>m</sub>_ over _F_
* a target polynomial _t_ over _F_
* an injective (preserves distinctness) function _f_ from inputs (index _i_ and the corresponding input bit _u\[i]_) to polynomial indices (between 1 and m)

The task is to write _t_ as a linear combination of the basis polynomials. For each input _u_, the function _f_ restricts the polynomials that can be used : an input _u_ is accepted if there are tuples _a = (a<sub>1</sub> ,..., a<sub>m</sub>)_ and _b = (b<sub>1</sub> ,..., b<sub>m</sub>)_ from the field _F_ such that:
   * _a<sub>k</sub> = b<sub>k</sub> = 1_ if _k = f( i, u\[i] )_ for some _i_
   * _a<sub>k</sub> = b<sub>k</sub> = 0_ if _k = f( i, 1 - u\[i] )_ for some _i_
   * _t_ divides _v<sub>a</sub> • w<sub>b</sub>_ where _v<sub>a</sub> = Σ<sub>i</sub> a<sub>i</sub> • v<sub>i</sub>_ and _w<sub>b</sub> = Σ<sub>i</sub> b<sub>i</sub> • w<sub>i</sub>_
      * Note: the blog post has some indexing errors and it is not clear whether _v<sub>a</sub>_ and _w<sub>b</sub>_ also include functions _v<sub>0</sub>_ and _w<sub>0</sub>_, which are always present for every input. I think they just made a mistake and just intended to use 1-up indexing consistently

Expressing this as an algorithm:
   * For each input bit
      * call _f_ with the bit and its index. The result will be some index _m_. You must use _v<sub>m</sub>_ and _w<sub>m</sub>_ in the linear combination
      * call _f_ with the negated bit and its index. The result will be some index _m_. You cannot use _v<sub>m</sub>_ and _w<sub>m</sub>_ in the linear combination
   * Since _f_ is injective, these outputs will never overlap. You will have chosen _2•n_ indices and either selected or rejected the corresponding polynomials
   * If _m_ is greater than _2•n_, there will be some polynomials left over. You may choose to use them (and you can make that decision independently for the _v_ and _w_ components, which is why _a_ and _b_ are distinct tuples).
   * If its possible to choose polynomials such that _v • w_ is a multiple of _t_, the input _u_ is accepted by the QSP.
   
The values _a_ and _b_ are the witness, and the verifier simply needs to check that _t_ divides _v<sub>a</sub> • w<sub>b</sub>_, which can be done in polynomial time

#### Reduction from generic computations

Since QSP is **NP**-complete (not shown here), it is possible to reduce a generic computations to a QSP. All of the engineering/design work is ensuring the reduction produces a reasonable QSP.

#### Efficiency

1. Instead of requiring the prover to verify that _t_ divides _v<sub>a</sub> • w<sub>b</sub>_, the verifier can also supply the multiple _h_. Then the verifier only needs to check that _t • h - v<sub>a</sub> • w<sub>b</sub> = 0_
1. For large polynomials, even multiplying them together is too expensive. Instead, the verifier can choose a secret random point _s_ and only verify the equation at that point, which would only involve field operations.

This is obviously a weaker constraint, but since the prover doesn't know _s_ and the number of zeros (the degree of the polynomial) is tiny compared to the possible values of _s_ (the number of field elements), this is safe in practice.
   
### The zkSNARK process

##### Setup

* choose a group and generator _g_
* define the encryption operation as _E(x) := g<sup>x</sup>_
* choose a secret element _s_ and publish _E(s<sup>0</sup>), E(s<sup>1</sup>), ..., E(s<sup>d</sup>)_ where _d_ is the maximum degree of all polynomials
* choose another secret element _α_ and publish _E(α•s<sup>0</sup>), E(α•s<sup>1</sup>), ..., E(α•s<sup>d</sup>)_
* publish the evaluated polynomials _E(t(s)), E(α•t(s)), E(v<sub>0</sub>(s)), ..., E(v<sub>m</sub>(s)), E(α•v<sub>0</sub>(s)), E(α•v<sub>m</sub>(s)), E(w<sub>0</sub>(s)), ..., E(w<sub>m</sub>(s)), E(α•w<sub>0</sub>(s)), E(α•w<sub>m</sub>(s))_
* choose three secret numbers β<sub>v</sub>, β<sub>w</sub>, γ and publish _E(γ), E(β<sub>v</sub>•γ), E(β<sub>w</sub>•γ), E(β<sub>v</sub>•v<sub>1</sub>(s)), ..., E(β<sub>v</sub>•v<sub>m</sub>(s)), E(β<sub>w</sub>•w<sub>1</sub>(s)), ..., E(β<sub>w</sub>•w<sub>m</sub>(s)), E(β<sub>v</sub>•t(s)), E(β<sub>w</sub>•t(s))_
* forget all the secret values (this is the toxic waste in the ZCash protocol). Anyone who knows it will be able to generate fake proofs that happen to evaluate to zero at _s_

##### Evaluating a polynomial at a secret point

Note that _E_ has a homomorphic property that allows provers to compute _E(f(s))_ from the encrypted components:

* Let _f(x) = 4x<sup>2</sup> + 2x + 4_
* _E(f(s)) = E(4s<sup>2</sup> + 2s + 4) = g<sup>4s•s + 2s + 4</sup> = g<sup>4s•s</sup> • g<sup>2s</sup> • g<sup>4</sup> = E(s<sup>2</sup>)<sup>4</sup> • E(s<sup>1</sup>)<sup>2</sup> • E(s<sup>0</sup>)<sup>4</sup>_

The problem is that the verifier cannot check if the prover evaluated the polynomial correctly. To address this, the prover also publishes _E(α•f(s))_, computed in the same way. These two values can be compared for consistency using a pairing function with the property _e(g<sup>x</sup>, g<sup>y</sup>) = e(g, g)<sup>xy</sup>_ as follows:

_e( E(f(s)), E(α•s<sup>0</sup>) ) = e( g<sup>f(s)</sup>, g<sup>α</sup> ) = e(g, g)<sup>α•f(s)</sup> = e( g<sup>α•f(s)</sup>, g) = e( E(α•f(s)), g )_

A valid proof will pass this consistency check and it is believed impossible to produce a false positive, but this hasn't been proven. Note that this doesn't actually demonstrate that the prover evaluated _f(s)_, merely that they evaluated some polynomial at _s_. The actual zkSNARK will need to do an additional check to confirm the correct polynomial was used.

##### Computation in encrypted space

It is worth taking a second to recognize our homomorphic computation abilities:
* We can add encrypted values together as many times as we want ( since _E(x)*E(y) = E(x+y)_ )
* We can multiply an encrypted value by a scalar ( since _E(x)<sup>a</sup> = E(a•x)_ )
* We have precomputed the encrypted versions of various values, which can be combined using the above two properties
* We can use the pairing function to compare the prodcut of two encrypted values to the product of another two encrypted values (since if _pq = uv_ then _e(E(p), E(q)) = e(E(u), E(v))_ )


##### Generating a proof

Recall that when generating a proof, the injective function defines the coefficients of _2•n_ terms in the _v_ and _w_ polynomials. Define _v<sub>free</sub>(x) := Σ<sub>k</sub> a<sub>k</sub> • v<sub>k</sub>(x)_ where _k_ ranges over all the free indices.

The proof consists of:
* _V<sub>free</sub> := E(v<sub>free</sub>(s))_ and _V'<sub>free</sub> := E(α•v<sub>free</sub>(s))_
* _W := E(w(s))_ and _W' := E(α•w(s))_
* _H := E(h(s))_ and _H' := E(α•h(s))_
* _Y := E( β<sub>v</sub>•v<sub>free</sub>(s) + β<sub>w</sub>•w(s) )_

Note that the pairing function can be used to confirm that _V<sub>free</sub>_, _W_ and _H_ are all the encrypted versions of polynomials evaluated at _s_.


### An example

I want to prove that I know a solution to the equation

_x<sup>3</sup> + x + 5 = 35</sup>_ 

#### Flattening

_sym_1 := x * x_  \[ **x<sup>2</sup>** ]

_y := sym_1 * x_  \[ **x<sup>3</sup>** ]

_sym_2 := y + x_  \[ **x<sup>3</sup> + x** ]

_~out := sym_2 + 5_ \[ **x<sup>3</sup> + x + 5** ]

#### R1CS

We want to convert this into a rank-1 constraint system (R1CS): a set of constraints represented by vectors (_a_, _b_, _c_). The solution _s_ is a vector that satisfies all constraints with the equation _s•a * s•b = s•c_

In our example, _s_ represents coefficients for the variables 

\[ _~one, x, ~out, sym_1, y, sym2_ ]

where _~one_ represents the number 1 and all other variables are defined in the Flattening section.

The first constraint ( _sym_1 := x * x_ ) can be represented by the vectors:

_a<sub>0</sub> = \[0, 1, 0, 0, 0, 0]_

_b<sub>0</sub> = \[0, 1, 0, 0, 0, 0]_

_c<sub>0</sub> = \[0, 0, 0, 1, 0, 0]_

Similarly, ( _y := sym_1 * x_ ) is represented as:

_a<sub>1</sub> = \[0, 0, 0, 1, 0, 0]_

_b<sub>1</sub> = \[0, 1, 0, 0, 0, 0]_

_c<sub>1</sub> = \[0, 0, 0, 0, 1, 0]_

_sym_2 := y + x_ is represented as:

_a<sub>2</sub> = \[0, 1, 0, 0, 1, 0]_

_b<sub>2</sub> = \[1, 0, 0, 0, 0, 0]_

_c<sub>2</sub> = \[0, 0, 0, 0, 0, 1]_

_~out := sym_2 + 5_ is represented as:

_a<sub>3</sub> = \[5, 0, 0, 0, 0, 1]_

_b<sub>3</sub> = \[1, 0, 0, 0, 0, 0]_

_c<sub>3</sub> = \[0, 0, 1, 0, 0, 0]_

The complete R1CS:

A = 

_\[0, 1, 0, 0, 0, 0]_

_\[0, 0, 0, 1, 0, 0]_

_\[0, 1, 0, 0, 1, 0]_

_\[5, 0, 0, 0, 0, 1]_

B = 

_\[0, 1, 0, 0, 0, 0]_

_\[0, 1, 0, 0, 0, 0]_

_\[1, 0, 0, 0, 0, 0]_

_\[1, 0, 0, 0, 0, 0]_

C = 

_\[0, 0, 0, 1, 0, 0]_

_\[0, 0, 0, 0, 1, 0]_

_\[0, 0, 0, 0, 0, 1]_

_\[0, 0, 1, 0, 0, 0]_


A witness is any assignment that satisfies all constraints:

_S = \[1, 3, 35, 9, 27, 30]_

#### QAP

We now want to represent this using polynomials instead of dot products. In particular, we want the polynomials at each x coordinate to represent one constraint (ie. x=1 represents the first constraint).

* _A<sub>~one</sub>(x)_ passes through (1,0) (2,0) (3,0) (4,5) 
* _A<sub>x</sub>(x)_ passes through (1,1) (2,0) (3,1) (4,0) 
* _A<sub>~out</sub>(x)_ passes through (1,0) (2,0) (3,0) (4,0)
* etc

We can use Lagrange interpolation to construct the polynomials. For example, to construct a polynomial that passes through (1,3) (2,2) and (3,4):
* make a polynomial that passes through (1,3) (2,0) and (3,0):
   * f (x) = ( x - 2 )( x - 3 ) \[ a polynomial with zeroes at x = 2,3 ]
   * y = f(x) * 3 / f(1)  \[ scale it so it passes through (1,3) ]
* make a polynomial that passes through (1,0) (2,2) and (3,0) in the same way
* make a polynomial that passes through (1,0) (2,0) and (3,4) in the same way
* add them all together

The purpose of this transformation is that the polynomials now encode all of the constraints simultaneously. To see this, take the dot product of the witness _S_ with the vector of polynomials _A_:

_A•S = s<sub>~one</sub> A<sub>~one</sub> + s<sub>x</sub> A<sub>x</sub> + s<sub>~out</sub> A<sub>~out</sub> + s<sub>sym_1</sub> A<sub>sym_1</sub> + s<sub>y</sub> A<sub>y</sub> + s<sub>sym_2</sub> A<sub>sym_2</sub>_

This passes through (1, s<sub>x</sub>) (2, s<sub>sym_1</sub>) (3, s<sub>x</sub> + s<sub>y</sub>) and (4, 5s<sub>~one</sub> + s<sub>sym_2</sub>).

Similarly:
* _B•S_ passes through (1, s<sub>x</sub>) (2, s<sub>x</sub>) (3, s<sub>~one</sub>) and (4, s<sub>~one</sub>)
* _C•S_ passes through (1, s<sub>sym_1</sub>) (2, s<sub>y</sub>) (3, s<sub>sym_2</sub>) and (4, s<sub>~out</sub>)

This means that _t = A•S * B•S - C•S_ will pass through the points (1, s<sub>x</sub> s<sub>x</sub> - s<sub>sym_1</sub>) (2, s<sub>sym_1</sub> s<sub>x</sub> - s<sub>y</sub>) (3, s<sub>x</sub> s<sub>~one</sub> + s<sub>y</sub> s<sub>~one</sub> - s<sub>sym_2</sub>) and (4, 5s<sub>~one</sub> s<sub>~one</sub> + s<sub>sym_2</sub> s<sub>~one</sub> - s<sub>~out</sub>). If these points all evaluate to zero, then all of the original constraints are satisfied with the chosen witness _S_.

In practice, we don't actually evaluate _t_ at all integer points. Instead we construct a minimal polynomial that has zeros in all the right places:

_Z = (x - 1) * (x - 2) * (x - 3) * (x - 4)_

We now simply need to confirm that _t_ is a multiple of _Z_.

### Elliptic Curve Pairings

Elliptic Curves:
* Have points that form a group under a special "adding" operation, provided you include the _Point at infinity_. 
* There is an agreed generator point _G_

Pairings allow you to check some unexpected equations. For example, if _P = p*G_, _Q = q*G_ and _R = r*G_, you can use _P_, _Q_ and _R_ to check if _p*g=r_. Another way of viewing them is that pairings allow you to check quadratic constraints (in addition to the linear constraints that are trivial to check with normal elliptic curve arithmetic).

The pairing is also called a bilinear map, because it satisfies the constraints:

* _e(P, Q + R) = e(P, Q) * e(P, R)_
* _e(P + S, Q) = e(P,Q) * e(S, Q)_

#### Prime Field

A prime field is (homomorphic to) the set of numbers 0,1,2,...,_p_-1 where all the math is done modulo _p_. Division is accomplished by finding the inverse element (the value that multiplies with the original element to get 1): _b<sup>-1</sup> = b<sup> p - 2</sup>_

#### Extension Field

Extension fields work by taking an existing field and adding a new element (and defining the relationship between that new element and existing elements) and then defining the field elements as all linear combinations of the original field and the new element. Note that there are now _p<sup>2</sup> - 1_ elements in the multiplicative group so the inverse is: _b<sup>-1</sup> = b<sup> p**2 - 2</sup>_. With prime fields, we can make cubic extensions (where the mathematical relationship between some new element _w_ and existing field elements is a cubic equation, so 1, _w_ and _w**2_ are all linearly independent of each other.

#### Some notation

* _G1_ is an elliptic curve over _F<sub>p</sub>_
   * all points satisfy an equation of the form _y<sup>2</sup> = x<sup>3</sup> + b_
   * both coordinates are elements of _F<sub>p</sub>_
* _G2_ is an elliptic curve over _F<sub>p12</sub>_
   * all points satisfy the same equation as _G1_
   * the coordinates are elements of a degree 12 extension field
* _Gt_ is _F<sub>p12</sub>_

The pairing is a map from _G2 x G1 -> Gt_

Define a _divisor_: count the zeroes and infinities of a function. For example:
   * Fix some point _P = (P<sub>x</sub>, P<sub>y</sub>)_
   * Consider _f(x,y) = x - P<sub>x</sub>_
   * The divisor is _\[P] + \[-P] - 2 * \[O]_
      * the function equals zero at P (note that _x_ is _P<sub>x</sub>_)
      * the function equals zero at _-P_ since _-P_ and _P_ share the same _x_ coordinate
      * the function goes to infinity as _x_ goes to infinity. It's negative because it's an infinity and not a zero, and has a multiplicity of 2 for technical reasons having to do with the relative speed at which _x_ and _y_ approach infinity

Another example:
   * Consider a line function: _ax + by + c = 0_, where the lines passes through points _P_ and _Q_.
   * This implies it also passes through _-P-Q_ (because of how elliptic curve addition works)
   * It goes to infinity depending on both x and y (multiplicity of 3), so the divisor is _\[P] + \[Q] + \[-P-Q]- 3 * \[O]_
   
Every rational function uniquely corresponds to some divisor up to a multiplication by a constant. The divisor of _F * G_ is equal to the divisor of _F_ plus the divisor of _G_. There is a theorem that says that if you "remove the square brackets" from a divisor of a function, the points must add to _O_.

#### Tate pairing

Consider the following functions defined via their divisors:
* _F<sub>P</sub> = n * \[P] - n * \[O]_, where _n_ is the order of _G1_
* _F<sub>Q</sub> = n * \[Q] - n * \[O]_
* _g = \[P + Q] - \[P] - \[Q] + \[O]_

If you multiply all functions together, the divisor simplifies to _n * \[P+Q] - n * \[O]_, which is the divisor for _F<sub>P + Q</sub>_ (so it's the same function up to a constant).

The "final exponentiation" step is to take the result of the function and raise it to the power of _z= (p<sup>12</sup> - 1) / n_. 

### References

* [An excellent article](https://blog.ethereum.org/2016/12/05/zksnarks-in-a-nutshell/) that heavily informed this research.

* [Another article by Vitalik](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)
