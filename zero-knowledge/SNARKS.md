
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


## Another article

The process of working through these articles has given me some intuition, but not enough that I can explain it clearly to someone without following the exact same steps. Let's summarize [this article](https://blockgeeks.com/guides/what-is-zksnarks/), which seems to be at a higher level. Perhaps this will cement my thinking.


### Properties of a zero-knowledge proof
* Completeness: an honest prover can convince an honest verifier of a true statement
* Soundness: a dishonest prover cannot convince an honest verifier of a false statement
* Zero knowledge: the verifier cannot learn anything other than the statement is true

#### Trivial Examples
* They explain the Alibaba cave, waldo and sudoku examples. I already understand them so I won't bother summarizing

#### What are we proving
* There is a difference between proving a statement and proving knowledge of a value.
* In the blockchain world we care about proving knowledge (specifically, the knowledge of a private key)


### Schnorr Identification Protocol
* Anna wants to prove that she knows the private key `s` associated with a given public key `v`.
* Everyone knows
   * a prime `p`, 
   * `q`, a factor of `p-1`
   * a generator `a` with cycle length `q`
* Anna claims to know `s`
* v = a<sup>-s</sup> mod q

* Anna encrypts a random `r`:
   * X = ar mod p
* She calculates a value that depends on this and a message `M`:
   * e = H ( M || X)
* She calculates a signature that depends on `r`, `s` and `e`:
   * y = r + s * e mod q
* She sends `M`, `e` and `y` to Carl
   * since `r` is private, this does not reveal `s`
* Carl can compute:
   * X' = a<sup>y</sup> * v<sup>e</sup><br>
        = a<sup>r</sup>a<sup>se</sup>a<sup>-se</sup><br>
        = X
* This only works because Anna was able to produce `y`, which depends on knowing `s`

### Fiat-Shamir
* There is a group with generator `g` (mod p is implied for the rest of this section)
* Anna wants to prove she knows `x` such that <code>y = g<sup>x</sup></code> 
* She picks a random `v` and computes <code>t =  g<sup>v</sup></code> 
* Carl picks a random challenge `c`
* Anna computes `r = v - c*x` and gives `r` to Carl
* Carl checks if <code>t == g<sup>r</sup> * y<sup>c</sup></code>

I think this will be clearer if we distinguish <span style="color:red">encrypted values</span> from <span style="color:green">plaintext values</span>:
* Recall, within the encrypted space, anyone can add values and multiply by scalars
* Everyone knows <span style="color:red">x</span> and Anna wants to prove she knows <span style="color:green">x</span>
* Anna generates a random <span style="color:green">v</span> and sends Carl <span style="color:red">v</span>
* Carl sends Anna <span style="color:green">c</span>
* At this point, Anna can calculate <span style="color:green">v - cx</span> iff she knows <span style="color:green">x</span>. Carl can do the same calculation in the encrypted space <span style="color:red">v - cx</span>
* Anna returns the result <span style="color:green">v - cx</span> and Carl confirms it encrypts to his calculation

#### Making it non-interactive
* This protocol relies on Carl choosing a value that Anna could not predict. The implication seems to be that premature knowledge of <span style="color:green">c</span> would let her choose  <span style="color:green">v</span> in a way that doesn't require knowledge of <span style="color:green">x</span>. I haven't been able to figure out the attack yet.
* We could make this non-interactive by having Anna use the hash of (g, <span style="color:red">x</span> and <span style="color:red">v</span>). This ensures she chose `v` before `c`

### zkSNARKS
* The rest of the article explains zkSNARKs by just asserting the existence of proving and verifying algorithms. There is no information here to be learned.

## The ZCash explainer series

Here is [another resource that seems to have some details](https://electriccoin.co/blog/snark-explain/)

### Homomorphic Hidings

This is just making the same point about the ability to perform operations on encrypted data.
* It is possible to calculate E(x+y) from E(x) and E(y)
* This follows directly if we use the discrete log problem as the encryption operation c = g<sup>x</sup> mod p
* Then E(x)•E(y) = g<sup>x</sup>g<sup>y</sup> = g<sup>x+y</sup>
* It is also possible to multiply by a scalar: E(x)<sup>c</sup> = g<sup>cx</sup> = E(cx)

### Blind Evaluation of polynomials
* Suppose:
   * Alice knows a polynomial P defined over a field F and Bob knows a point in the field s
   * Bob wants to learn E(P(s)) without revealing s (and Alice will not reveal P)
* They can solve this problem with the homomorphic hidings:
   * Bob sends Alice E(1), E(s), E(s<sup>2</sup>), ... , E(s<sup>d</sup>). Note that this is the encrypted version of all x<sup>n</sup> evaluated at s.
   * Alice uses the homomorphic hiding property to calculate E(P(s)) and sends it to Bob.
* The general idea is that Bob is a verifier with a "correct" polynomial in mind, and he wants to check if Alice knows it. If she doesn't know it, she will be unable to produce E(P(s)). We don't want Alice to send P directly to Bob because it's very large. Of course, the set of E(s<sup>n</sup>) values that Bob sends is just as large but in practice we hard-code it into the protocol. 

### Knowledge of Coefficient Test
* The way I described the protocol implies that Bob knows P, so he can just compare the E(P(s)) value with one he calculates himself
* This section of the blog claims that Bob can't tell if Alice sends him something other that E(P(s)). I'm not sure what the resolution is, but let's go with it for now. It is probably something about Bob having a "correct" polynomial in mind is not the same as saying he actually knows the polynomial.
* We are going to build a Knowledge of Coefficient (KC) Test to help confirm Alice is following the protocol.
* Notational conventions:
   * we will describe our group additively from now on:
       * our encryption is now: E(x) = x⋅g 
       * E(x) + E(y) = x⋅g + y⋅g = (x+y)⋅g = E(x+y)
       * cE(x) = cx⋅g = E(cx)
   * a pair of group elements (a, α⋅a) is an α-pair (assuming neither value is zero)
* The KC test
   * Bob chooses a random a and α and sends (a, α⋅a) to Alice
   * Alice must respond with a different α-pair
      * note: she doesn't know α so she can't pick a new pair directly
      * she can pick some non-zero γ and respond with Bob's pair, with each coordinate multiplied by γ ie. (γ⋅a, γ⋅α⋅a)
   * Since Bob knows α he can check this directly
   * Bob will assume that Alice calculated the value from his own value. In other words, he believes that Alice knows γ

### Verifiable blind polynomial evaluation
* We want to recreate the "blind evaluation of a polynomial" scheme but also prevent Alice from responding with anything other than E(P(s))
* In the original KC test, Alice had to derive an α-pair from the one that Bob sent
* What if Bob sends several α-pairs. Can she use the pairs to derive another one? Yes, any linear combination of the pairs will produce another pair. Note that this is a more general version of the original process.
* As before, Bob will assume that this is how Alice was able to produce a pair - he now believes that Alice knows the linear combination of his original pairs.

The protocol:
* Bob sends Alice E(1), E(s), E(s<sup>2</sup>), ..., E(s<sup>d</sup>)  \[ in practice these are hardcoded in the protocol ]
* Bob chooses a random α and sends Alice E(α), E(αs), E(αs<sup>2</sup>), ..., E(αs<sup>d</sup>)
* Note: this is the same as saying, Bob gives Alice a bunch of α-pairs (E(1), αE(1)), (E(s), αE(s)), ... (E(s<sup>d</sup>), αE(s<sup>d</sup>))
* Alice uses these values to compute E(P(s)) and E(αP(s)) and sends both to Bob
* Bob checks that they are an α-pair
* Bob now believes that Alice responded with some linear combination of the original pairs he sent. Since those values were the hidings of the powers of s, this means she responded with a hiding of a polynomial in s.
* **Note: as far as I can tell, we're not saying anything about what the particular polynomial is**


### Polynomials representing computations
* This is the same QAP discussion we've already seen but it uses different terminology
* Personally, I find Vitalik's post easier to understand but I'll go through the example anyway to cement the idea

We would like to translate a computation into a Quadratic Arithmetic Program (QAP), which is a set of polynomials in multiple variables. Each polynomial is at most quadratic in all the variables. This constraint can be enforced by describing the program as a series of gates that combine two variables with either addition or multiplication.

For example, let's assume Alice wants to prove she knows three constants c that satisfy the equation:
(c<sub>1</sub>⋅c<sub>2</sub>)(c<sub>1</sub> + c<sub>3</sub>) = 7

This corresponds to the constraints:
* c<sub>4</sub> = c<sub>1</sub>⋅c<sub>2</sub>
* c<sub>5</sub> = c<sub>4</sub>⋅(c<sub>1</sub> + c<sub>3</sub>)

Note that each constraint has exactly one multiplication. We can think of each constraint as a gate with three parts: 
1. the input to the left of the multiplication
1. the input to the right of the multiplication
1. the output 


Alice needs to prove that she knows values (c<sub>1</sub>,...,c<sub>5</sub>) such that c<sub>5</sub>=7. Note that this external requirement (that c<sub>5</sub>=7) can be modeled as an additional constraint.

#### Reduction to QAP

We associate each constraint with a field element (the first equation is associated with the number 1, the second equation with the number 2). The elements {1,2} are our target points.

We now produce a set of polynomials that are either 0 or 1 on the target points. Each polynomial corresponds to one of the variables and one component (left, right or output) and the values at the target points correspond to whether it matches the constraint:
* Constraint one matches the function 2 - X  (it is 1 at target point 1 and 0 at target point 2)
   *  L<sub>1</sub> = R<sub>2</sub> = O<sub>4</sub> = 2 - X
   * c<sub>1</sub> is on the left, c<sub>2</sub> is on the right, c<sub>4</sub> is the output of constraint 1
* Constraint two matches the function X - 1 (it is 0 at target point 1 and 1 at target point 2)
   * L<sub>4</sub> = R<sub>1</sub> = R<sub>3</sub> = O<sub>5</sub> = X - 1
   * c<sub>4</sub> is on the left, c<sub>1</sub> and c<sub>3 </sub> are on the right, c<sub>5</sub> is the output of constraint 2
* All other functions are zero
   * eg. c<sub>2</sub> is not on the left of either constraint so L<sub>2</sub>(1) = L<sub>2</sub>(2) = 0

Now we can define:
* L = ∑<sub>i</sub> c<sub>i</sub>⋅L<sub>i</sub> = c<sub>1</sub>⋅(2-X) + c<sub>4</sub>⋅(X-1)
* R = ∑<sub>i</sub> c<sub>i</sub>⋅R<sub>i</sub> = c<sub>1</sub>⋅(X-1) + c<sub>2</sub>⋅(2-X) + c<sub>3</sub>⋅(X-1)
* O = ∑<sub>i</sub> c<sub>i</sub>⋅O<sub>i</sub> = c<sub>4</sub>⋅(2-X) + c<sub>5</sub>⋅(X-1)

The polynomial that captures all of these constraints is:
P = L⋅R - O

Note that each target point reproduces the original constraints
* P(1) = c<sub>1</sub>⋅c<sub>2</sub> - c<sub>4</sub>
* P(2) = c<sub>4</sub>⋅(c<sub>1</sub> + c<sub>3</sub>) - c<sub>5</sub>

So P has zeroes on the target point iff c<sub>1</sub> to c<sub>5</sub> are set correctly.

If we factor P, each factor corresponds to one of the zeroes. So we can define the target polynomial as one that zeroes all target points: T(X) = (X - 1)⋅(X - 2). This will divide P iff c<sub>1</sub> to c<sub>5</sub> are set correctly.

### The Pinocchio Protocol

Recall, if Alice knows a satisfying assignment for the c values, then P can be factored into polynomials H and T. 
Naturally, this equality will hold at any x coordinate `s`: P(s) = H(s)⋅T(s). On the other hand, if Alice does not know a satisfying assignment (and `p` is much larger than 2d, the maximum degree of P) this equality will almost certainly not hold at a randomly chosen x coordinate.

So the protocol is:
1. Alice chooses the polynomials L, R, O and H of degree `d` at most
1. Bob chooses a random point s and computes E(T(s))
1. Alice sends Bob the hidings of all these polynomials evaluated at `s`  E(L(s)), E(R(s)), E(O(s)), E(H(s))
1. Bob checks if E(L(s)⋅R(s) - O(s)) = E(T(s)⋅H(s))  \[note: we will discuss how Bob can check an encrypted multiplication when we discuss elliptic curve pairings]

The verifiable blind polynomial evaluation protocol allows Alice to perform step 3, but it doesn't confirm that the polynomials came from a valid assignment of c values. If she doesn't know a valid assignment, she can still produce unrelated polynomials L, R, O and H that will pass the final check. We need to confirm that the coefficients of L are the same as the coefficients of R, O and H. 


Let's combine the polynomials into one polynomial:

F = L + X<sup>d+1</sup>⋅R + X<sup>2(d+1)</sup>⋅O

We can do the same trick for the individual polynomials in the QAP:

F<sub>i</sub> = L<sub>i</sub> + X<sup>d+1</sup>⋅R<sub>i</sub> + X<sup>2(d+1)</sup>⋅O<sub>i</sub>

Note that if we sum two of the F<sub>i</sub> values, the individual polynomials sum separately:

F<sub>1</sub> + F<sub>2</sub> = L<sub>1</sub> + L<sub>2</sub> + X<sup>d+1</sup>⋅(R<sub>1</sub> + R<sub>2</sub>) + X<sup>2(d+1)</sup>⋅(O<sub>1</sub> + O<sub>2</sub>)

So if F is a linear combination of the F<sub>i</sub> values, the same coefficients will be used for each of the individual polynomials, which would prove they were produces from an assignment (Bob still needs to check that the last equality holds to prove it is a valid assignment)

So Bob chooses a random β and sends Alice the hidings E(β), E(βF<sup>1</sup>(s)), E(βF<sup>2</sup>(s)), ..., E(βF<sup>m</sup>(s)). If Alice returns a β-pairing she has written F as a linear combination of the F<sub>i</sub> values.

#### Adding zero knowledge

The scheme described does leak some information. At the very least, Bob can check if a putative set of satisfying constants matches the ones that Alice chose.

This can be solved easily by adding a random T-shift to each polynomial:
* L<sub>z</sub> = L + δ<sub>1</sub>⋅T for a random δ<sub>1</sub>
* R<sub>z</sub> = R + δ<sub>2</sub>⋅T for a random δ<sub>2</sub>
* O<sub>z</sub> = O + δ<sub>3</sub>⋅T for a random δ<sub>3</sub>

Since we have added T to every term, if T used to divide L⋅R - O, it will now divide L<sub>z</sub>⋅R<sub>z</sub> - O<sub>z</sub>. The other factor H will also need to be adjusted accordingly, but the proof will still work. However, each polynomial now has a random shift, so no information is revealed.


### Elliptic Curves and their pairings
* This article explains what an elliptic curve group is. I won't bother summarizing it here.
* Let's denote the group over F<sub>p</sub> (the field of integers mod p) to be G<sub>1</sub>. We will restrict ourselves to situations where G<sub>1</sub> has prime size r
* The smallest integer k such that r divides p<sup>k</sup> - 1 is the embedding degree of the curve. If it is large enough, the discrete log problem in G<sub>1</sub> is hard.
* The curve points over the field F<sub>p^k</sub> also form a group. The group clearly contains G<sub>1</sub>. It also contains additional subgroups of order r. Let's select one and call it G<sub>2</sub>.
* The **multiplicative** group of F<sub>p^k</sub> also contains a subgroup of order r, which we will denote G<sub>T</sub>

* Let's pick generators g and h for G<sub>1</sub> and G<sub>2</sub> respectively. Here we assert the existence of an efficient map called the Tate reduced pairing, which maps a pair of elements from G<sub>1</sub> and G<sub>2</sub> into an element of G<sub>T</sub> with the properties:
   * Tate(g, h) = **g** for a generator **g** of G<sub>T</sub> (we can map the generators to another generator)
   * given F<sub>r</sub> elements a,b, Tate(a⋅g, b⋅h) = **g**<sup>ab</sup>.

So we can define three hidings that support addition and multiplication by a scalar (the same way we've already done it):
* E<sub>1</sub>(x) = x⋅g
* E<sub>2</sub>(x) = x⋅h
* E(x) = x⋅**g**
The Tate pairing gives us one additional power:
* Given E<sub>1</sub>(x) and E<sub>2</sub>(y), we can compute E(xy). Since these are different groups, we can't perform the operation repeatedly. Instead, we have a one-time move to produce a hiding of the product of two hidden values.

### Non-interactive evaluation
* The protocol that we've described asks Bob to send Alice various hidings.
* Ideally, Alice should be able to produce a proof without interaction from Bob.
* Instead, we settle for a trusted setup procedure, where Bob produces all the hidings once and then publishes them for everyone to use in their own proofs. Importantly, the proofs rely on the assumption that Bob keeps the s, α, β values private. If we are simply going to publish the same set of values for everyone, then it is important that the secret values are destroyed (the toxic waste in the ZCash protocol). In fact, in ZCash, those values (and the hidings) where produced in a multi-party computation to minimize the chance that anyone knows those values.
* We refer to the shared set of hidings as a Common Reference String (CSR).  

* Now we can modify the polynomial evaluation protocol to be non-interactive:
   * Bob originally sends Alice E(1), E(s), E(s<sup>2</sup>), ..., E(s<sup>d</sup>) and E(α), E(αs), E(αs<sup>2</sup>), ..., E(αs<sup>d</sup>) for a random α. These are now hardcoded in the CSR so we can skip this step.
   * Alice computes E(P(s)) and E(αP(s))
   * Bob originally verifies this by multiplying the first value by α (within the hiding). Now we're trying to prove this to a generic person who doesn't know α. 
   * We will use the Tate pairing. In this entire protocol:
      * whenever someone is hiding the left hand side of an α-pair, they should use the E<sub>1</sub> hiding function
      * whenever someone is hiding the right hand side of an α-pair, they should use the E<sub>2</sub> hiding function
      * Bob wants to confirm that E<sub>1</sub>(P(s)) E<sub>2</sub>(αP(s)) are the hidings of an α-pair. He computes E(αP(s)) in two different ways and sees if they match:
         * Tate(E<sub>1</sub>(P(s)), E<sub>2</sub>(α))
         * Tate(E<sub>1</sub>(1), E<sub>2</sub>(αP(s)))
      
