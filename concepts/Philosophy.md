
# Cryptographic Philosophy

## Radical asymmetry

Consider the apparent paradox: nobody would store $10,000 in a safe they bought for $100 yet people freely access their bank accounts **over the internet** using laptops worth $100. This contrast is particularly acute in the blockchain space, where data is readily accessible, extraordinarily valuable and yet still alleged to be secure against state-level actors. How can consumer-grade devices possibly be resilient to potential adversaries with billions of dollars? The answer is cryptography.

Modern cryptography is a radically asymmetric tool - it is exponentially harder to attack than use. To understand this claim intuitively, not just at an intellectual level, we will explore a simple model. Consider a storage unit protected by a three-digit combination lock.

> ? - ? - ?

Whoever knows the combination can unlock it in seconds. Anyone else will have to try all possible combinations until they happen to guess correctly, which would require 500 tries on average and 1000 in the worst case. This will take several minutes. In other words, it is much easier to use the lock in the intended way than to misuse it. This asymmetry is true of any sensible security system in the physical or digital world, but mathematical security is interesting because of how it scales.

Adding an extra digit to the combination will only increase the unlock time by an extra second, but an attacker now has 10000 combinations to try, which will take hours.

> ? - ? - ? - ?

Add one more digit and they will require days. Notice that each extra digit increases the attack time by a factor of 10, which leads to accelerating returns - the more digits in the combination, the greater the security provided by each additional digit.

> ? - ? - ? - ? - ?

You can see where this is going but to get up to cryptographic strengths, adding one digit at a time will take far too long. Let's double the number of digits to ten.

> ? - ? - ? - ? - ? - ? - ? - ? - ? - ?

This would be annoying to use, but it would still take less than a minute. Take a moment to imagine trying every possible 10-digit number. The attacker (who we will generously assume can try one combination per second) will take over 300 years. Double the length to 20 digits, and it will take the attacker more than 3 trillion years (about 200 times longer than the current age of the universe).

On the other hand, I'm using the lock combination as an analogy for a cryptographic key, which is just a random number. When we transition from physical locks to cryptographic keys, we have to consider the possibility of parallel computing. If some attacker with massive resources (and nothing better to do) obtained a supercomputing cluster with a million nodes that can each try a trillion combinations per second, they could try every combination in a couple of minutes.

Luckily, we can just double the key length to 40 digits and it will take them over 300 trillion years. Double it once more and we have just exceeded the strength of a Bitcoin or Ethereum private key (which is 256 bits, approximately equivalent to 77 digits).

The important point is that anyone can trivially generate a random 256-bit number that will be secure from brute-force exhaustion against any adversary, no matter how powerful or well-resourced.

## Designing cryptographic systems

The challenge is to embed this feature into a complete security system. After all, a motivated adversary would not surrender at the sight of a 40-digit combination lock. They couldn't check all combinations but they could attempt to:
* break the lock
* drill a hole in the storage unit
* review the lock schematics for bugs, or even better, tamper with it in the supply chain
* listen or feel for signs that a given digit is in the correct position
* analyze fingerprints on the lock to deduce the combination
* secretly videotape the user inputting the combination
* guess combinations likely to be chosen by the user
* execute the [wrench attack](https://xkcd.com/538/)
* access the storage unit when it is unlocked
* find where the user wrote down the combination
* trick the user into attempting to open a different lock
* etc

Any one of these would undermine the radical asymmetry provided by the lock. In the world of cryptography, the primitives (hash functions, ciphers, signature algorithms) each have a specific well-defined purpose, and the analogues of these attacks are all instances of:
1. leveraging inconsistencies between the design and implementation
1. exploiting side-effects of the implementation
1. covertly replacing the software or hardware to change the functionality
1. breaking the expected guarantees of the primitive
1. bypassing the cryptographic primitive
1. obtaining the secret key (if it exists)

Let's discuss the cryptographic considerations of each of these attacks.

### Implementation Failures

The first three categories are examples of implementation or deployment vulnerabilities that apply to generic hardware and software. They will not be considered in this article, which focuses on the philosophical principles of cryptography, except to note that they cannot be mitigated with radical asymmetry. This means that powerful adversaries could plausibly exploit them. It is worth mentioning that public open-source blockchains that can tolerate malicious actors have some natural defences. Nevertheless, we will always require a strong security community to prevent, identify and mitigate issues as they arise.

### Breaking the primitive

Fortunately, there are a range of strong and well-studied cryptographic primitives with known security properties. Since cryptography is built on mathematics, a possible misconception is that cryptographic primitives are provably secure. In fact, with the exception of the impractical one-time-pad cipher, they actually rely on **computational security**. This just means that the best known attack algorithms run in exponential time, and are therefore also vulnerable to radical asymmetry.

Consequently, to mitigate this attack, the security architect simply needs to choose the algorithm parameters and key sizes such that attempting to break the primitive will exceed the resources of any plausible adversary.

### Bypassing the cryptographic primitive

This is straightforwardly mitigated by good design.

For example, one common security principle is **Defence in Depth**. This means that an adversary would need to bypass multiple mechanisms (a firewall, intrusion detection systems, air-gaps, user permissions, etc) before they can compromise a system. However, each mechanism is independent, which means it can be overcome individually. This implies that chaining security techniques cannot provide radical asymmetry, no matter how many hurdles are in place, and should not be used as an alternative to the cryptographic mechanism.

Similarly, another principle is **Security through Obscurity**. This could be as simple as using discretion when discussing details about a security system or as sophisticated as steganography (hiding messages within other messages). In all cases, it relies on the adversary lacking knowledge about the system. This is extremely fragile and cannot scale, since all users need to know the secret, and each confidant introduces a potential vulnerability. Cryptographic systems, on the other hand, are much more robust since they protect against adversaries who know everything except the deployment-specific secrets, which allows them to be widely studied and used publicly.

In some cases, secure systems require a mechanism to bypass the cryptography. This may occur, for example, as part of a recovery procedure when a user loses their cryptographic key. In such cases, the alternative paths should also use radical asymmetry with an equivalent amount of computational security.

The goal is to ensure the adversary has to overcome the radical asymmetry at some point.

### Obtaining the secret key

The central theme of this article is that a well-designed cryptographic system creates an insurmountable asymmetry between legitimate and illegitimate users. In most cases, legitimate users are identified through private cryptographic keys, which means that they are in control of their security. This is an excellent security property but can also be fragile, since [anyone who obtains a copy of the private key gets full access to the corresponding privileges](https://www.youtube.com/watch?v=dnC5mFaIW3Q).

Fortunately, in many designs the production and handling of the secret keys are relegated to an independent, upgradeable and reusable key management system that can be configured to the sophistication of the users, the available technology and the desired trade-off between security and fragility. Although it is crucial for security, we can often defer the topic of key management when designing or analyzing cryptographic systems.
