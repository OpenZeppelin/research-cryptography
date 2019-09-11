## Properties of Hash Functions

* Fixed size output
* Pre-image resistance
* Second pre-image resistance
* Collision Resistance
   * Demi and Leo share a birthday!
* All properties apply to all degrees

## Hash Commitments

| Name | Hash | Reveal |
|------|------|--------|
| Austin | ca1210fe21a3543526e446d4cd43c21c2df52d7a0ce99b6e98f9905c5e593bff | echo -n "By the birthday paradox we can expect to find a collision after about  sqrt(2**256) = 2**128 attempts. Salt:f0 47 7c 55 46 49 da bc 08 aa 6b 30 1c 69 0b 29" \| shasum -a 256 |
| Austin (bonus) | 1d17932990e5966716159f392b09f22997b170d1d514c74869fca9776a7c34e0 |  echo -n "Because my answer may not have had much entropy. In that case someone could guess at my answer and check the hash to see if they were correct. By including a (large) random number in my answer I make it infeasible for someone to guess/check my answer. Salt:29 4e 74 21 bb 0a ee 3a a3 c0 b0 0a 64 74 e2 56" \| shasum -a 256 |
| Nikesh | ca1210fe21a3543526e446d4cd43c21c2df52d7a0ce99b6e98f9905c5e593bff |  echo -n "By the birthday paradox we can expect to find a collision after about  sqrt(2**256) = 2**128 attempts. Salt:f0 47 7c 55 46 49 da bc 08 aa 6b 30 1c 69 0b 29" \| shasum -a 256 |
| Eric | e46f22dfdad004ad7e3d9730084d79054edf6d57052973a529db53a617d80f59 | Breaking keccak256 via the birthday attack: initialize a mapping(called myMap) of uint256 hash -> uint256 preimage. Assume that all pre-image values =0 at initialization. For n from 0 to (2^256 -1): thisHash = keccak256(n); if myMap\[thisHash] == 0: myMap\[thisHash] = n; else: print("I found a collision! Preimages " + n + " and " + myMap\[hisHash] + "result in the same hash"); endif; end; |
| Rick | b172510ac8fa9f5d4235985b46d651b2633d1cc485fff84c1541b1c0aaa2975d | choose a random 2^128 input and compare their hash results till we find a collision. If not, repeat this process one more time. hashed with keccak256 |

## Keccak40
> ./keccak40 nikesh:165839728124604 == ./keccak40 nikesh:111833654041068

## Use Cases
* Passwords
* Signatures
* Pseudo-random number generators
* Hash commitments
* ... many more
