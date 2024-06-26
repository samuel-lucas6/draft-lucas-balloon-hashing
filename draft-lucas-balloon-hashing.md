---
title: "Balloon Hashing"
docname: draft-lucas-balloon-hashing-latest
category: info

ipr: trust200902
keyword: Internet-Draft
submissiontype: IRTF

stand_alone: yes
smart_quotes: yes
pi: [toc, sortrefs, symrefs]

venue:
  github: "samuel-lucas6/draft-lucas-balloon-hashing"
  latest: "https://samuel-lucas6.github.io/draft-lucas-balloon-hashing/draft-lucas-balloon-hashing.html"

author:
 -
    fullname: Samuel Lucas
    organization: Individual Contributor
    email: ietf.tree495@simplelogin.fr


normative:

  FIPS202:
    title: "FIPS PUB 202 - SHA-3 Standard: Permutation-Based Hash and Extendable-Output Functions"
    target: https://doi.org/10.6028/NIST.FIPS.202
    author:
      -
        org: National Institute of Standards and Technology
    date: 2015

informative:

  PM99:
    title: "A Future-Adaptable Password Scheme"
    rc: "Proceedings of the 1999 USENIX Annual Technical Conference"
    target: https://www.usenix.org/legacy/publications/library/proceedings/usenix99/provos/provos.pdf
    author:
      -
        ins: N. Provos
        name: Niels Provos
        org: The OpenBSD Project
      -
        ins: D. Mazières
        name: David Mazières
        org: The OpenBSD Project
    date: 1999

  BCS16:
    title: "Balloon Hashing: A Memory-Hard Function Providing Provable Protection Against Sequential Attacks"
    rc: "Cryptology ePrint Archive, Paper 2016/027"
    target: https://eprint.iacr.org/2016/027
    author:
      -
        ins: D. Boneh
        name: Dan Boneh
        org: Stanford University
      -
        ins: H. Corrigan-Gibbs
        name: Henry Corrigan-Gibbs
        org: Stanford University
      -
        ins: S. Schechter
        name: Stuart Schechter
        org: Microsoft Research
    date: 2016

  RD16:
    title: "Proof of Space from Stacked Expanders"
    rc: "Theory of Cryptography. TCC 2016. Lecture Notes in Computer Science(), vol 9985, pp. 262–285"
    target: https://doi.org/10.1007/978-3-662-53641-4_11
    author:
      -
        ins: L. Ren
        name: Ling Ren
        org: Massachusetts Institute of Technology
      -
        ins: S. Devadas
        name: Srinivas Devadas
        org: Massachusetts Institute of Technology
    date: 2016

  AB16:
    title: "Efficiently Computing Data-Independent Memory-Hard Functions"
    rc: "Advances in Cryptology – CRYPTO 2016. CRYPTO 2016. Lecture Notes in Computer Science(), vol 9815, pp. 241–271"
    target: https://doi.org/10.1007/978-3-662-53008-5_9
    author:
      -
        ins: J. Alwen
        name: Joël Alwen
        org: IST Austria
      -
        ins: J. Blocki
        name: Jeremiah Blocki
        org: Microsoft Research
    date: 2016

  AB17:
    title: "Towards Practical Attacks on Argon2i and Balloon Hashing"
    rc: "2017 IEEE European Symposium on Security and Privacy (EuroS&P), Paris, France, 2017, pp. 142-157"
    target: https://doi.org/10.1109/EuroSP.2017.47
    author:
      -
        ins: J. Alwen
        name: Joël Alwen
        org: IST Austria
      -
        ins: J. Blocki
        name: Jeremiah Blocki
        org: Purdue University
    date: 2017

  ABP17:
    title: "Depth-Robust Graphs and Their Cumulative Memory Complexity"
    rc: "Advances in Cryptology – EUROCRYPT 2017. EUROCRYPT 2017. Lecture Notes in Computer Science(), vol 10212, pp. 3–32"
    target: https://doi.org/10.1007/978-3-319-56617-7_1
    author:
      -
        ins: J. Alwen
        name: Joël Alwen
        org: IST Austria
      -
        ins: J. Blocki
        name: Jeremiah Blocki
        org: Purdue University
      -
        ins: K. Pietrzak
        name: Krzysztof Pietrzak
        org: IST Austria
    date: 2017

  RD17:
    title: "Bandwidth Hard Functions for ASIC Resistance"
    rc: "Theory of Cryptography. TCC 2017. Lecture Notes in Computer Science(), vol 10677, pp. 466–492"
    target: https://doi.org/10.1007/978-3-319-70500-2_16
    author:
      -
        ins: L. Ren
        name: Ling Ren
        org: Massachusetts Institute of Technology
      -
        ins: S. Devadas
        name: Srinivas Devadas
        org: Massachusetts Institute of Technology
    date: 2017

--- abstract

This document describes Balloon, a memory-hard function suitable for password hashing and password-based key derivation. It has proven memory-hardness properties, is resistant to cache-timing attacks, is easy to implement, and is built from any collision-resistant pseudorandom function (PRF), hash function, or extendable-output function (XOF).

--- middle

# Introduction

Balloon {{BCS16}} is a memory-hard password hashing and password-based key derivation function that was published shortly after the Password Hashing Competition (PHC), which recommended Argon2 {{?RFC9106}}. It has several advantages over prior password hashing algorithms:

- It has proven memory-hardness properties, making it resistant against sequential GPU/ASIC attacks. An adversary trying to save space pays a large penalty in computation time.
- It can be instantiated with any collision-resistant PRF, hash function, or XOF, making it a mode of operation for these existing algorithms. No new, unstudied primitives are required.
- It uses a password-independent memory access pattern, making it resistant to cache-timing attacks. This property is especially relevant in cloud computing environments where multiple users can share the same physical machine.
- It is intuitive to understand and easy to implement, which reduces the risk of implementation mistakes.

Unfortunately, the paper did not fully specify the algorithm nor provide guidance on parameters. Furthermore, the algorithm was not designed with key derivation in mind and had multiple variants.

This document rectifies these issues and more by specifying an encoding, preventing canonicalization attacks, improving domain separation, fixing the modulo bias, making delta a constant, treating Balloon and Balloon-M as one algorithm, adding support for key derivation, and improving the performance.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Throughout this document, "byte" refers to the same unit as "octet", namely an 8-bit sequence.

Operations:

- `x++`: incrementing the integer `x` by 1 after it has been used in a function.
- `a ^ b`: the bitwise XOR of `a` and `b`.
- `a % b`: the remainder when dividing `a` by `b`.
- `a || b`: the concatenation of `a` and `b`.
- `a[i]`: index `i` of array `a`.
- `a.Length`: the length of `a` in bytes.
- `a.Slice(i, l)`: the copy of `l` bytes from byte array `a`, starting at index `i`.
- `ByteArray(l)`: the creation of a new byte array with length `l`.
- `BlockArray(i, l)`: the creation of a new array of arrays containing `i` byte arrays, each with length `l`.
- `PRF(k, m)`: the output of a collision-resistant PRF (e.g. HMAC-SHA512 {{!RFC2104}}) with key `k` and message `m`, both byte arrays. To use a collision-resistant hash function with no key parameter (e.g. SHA-512 {{!RFC6234}}), you MUST perform prefix MAC and pad the key with zeros to the block size.
- `LE64(x)`: the little-endian encoding of unsigned 64-bit integer `x`.
- `ReadLE64(a)`: the conversion of byte array `a` into an unsigned, little-endian 64-bit integer.
- `ZeroPad(a, n)`: byte array `a` padded with zeros until it is `n` bytes long.
- `Ceiling(x)`: the floating point `x` rounded up to the nearest whole number.
- `UTF8(s)`: the UTF-8 encoding of string `s`.

Constants:

- `HASH_LEN`: the output length of the hash function in bytes. For an XOF, this is the minimum output length to obtain the maximum advertised security level. For example, a 256-bit output for an XOF targeting 128-bit security.
- `MAX_PASSWORD`: the maximum password length, which is 4294967295 bytes.
- `MAX_SALT`: the maximum salt length, which is 4294967295 bytes.
- `MIN_SPACECOST`: the minimum space cost, which is 1 as an integer.
- `MAX_SPACECOST`: the maximum space cost, which is 4294967296 as an integer.
- `MIN_TIMECOST`: the minimum time cost, which is 1 as an integer.
- `MAX_TIMECOST`: the maximum time cost, which is 16777215 as an integer.
- `MIN_PARALLELISM`: the minimum parallelism, which is 1 as an integer.
- `MAX_PARALLELISM`: the maximum parallelism, which is 16777215 as an integer.
- `MAX_LENGTH`: the maximum output length, which is 4294967295 as an integer.
- `MAX_PEPPER`: the maximum pepper length, which is 64 bytes.

# The Balloon Algorithm

~~~
Balloon(password, salt, spaceCost, timeCost, parallelism, length, pepper)
~~~

Balloon calls an internal function that provides memory hardness in a way that supports parallelism, which enables greater memory hardness without increasing the delay. The result of XORing the internal function outputs together is then used alongside user provided parameters for key derivation following the Extract-then-Expand paradigm.

Inputs:

- `password`: the password to be hashed, which MUST NOT be greater than `MAX_PASSWORD` bytes long.
- `salt`: the unique salt, which MUST NOT be greater than `MAX_SALT` bytes long.
- `spaceCost`: the memory size in blocks, which MUST be an integer between `MIN_SPACECOST` and `MAX_SPACECOST` that is a power of 2. A block is `HASH_LEN` bytes long.
- `timeCost`: the number of rounds, which MUST be an integer between `MIN_TIMECOST` and `MAX_TIMECOST`.
- `parallelism`: the number of CPU cores/internal function calls in parallel, which MUST be an integer between `MIN_PARALLELISM` and `MAX_PARALLELISM`.
- `length`: the length of the password hash/derived key in bytes, which MUST NOT be greater than `MAX_LENGTH`.
- `pepper`: an optional secret key, which MUST NOT be greater than `MAX_PEPPER` bytes long.

Outputs:

- The password hash/derived key, which is `length` bytes long.

Steps:

~~~
outputs = BlockArray(parallelism, HASH_LEN)

if pepper.Length == 0
    key = ZeroPad(ByteArray(0), HASH_LEN)
else
    key = pepper
key = PRF(key, password || salt || LE64(password.Length) || LE64(salt.Length))

parallel for i = 0 to parallelism - 1
    outputs[i] = BalloonCore(key, salt, spaceCost, timeCost, parallelism, length, i + 1)

hash = ByteArray(HASH_LEN)
foreach output in outputs
    for i = 0 to output.Length - 1
        hash[i] = hash[i] ^ output[i]

counter = 1
reps = Ceiling(length / HASH_LEN)
previous = ByteArray(0)
result = ByteArray(0)
for i = 0 to reps
    previous = PRF(key, previous || LE64(counter++) || UTF8("balloon") || hash)
    result = result || previous

return result.Slice(0, length)
~~~

# The BalloonCore Function

~~~
BalloonCore(key, salt, spaceCost, timeCost, parallelism, length, iteration)
~~~

The BalloonCore function is the internal function used by Balloon for memory hardness. It can be divided into three steps:

1. Expand: a large buffer is filled with pseudorandom bytes derived by repeatedly hashing the user provided parameters. This buffer is divided into blocks the size of the hash function output length.
2. Mix: the buffer is mixed for the number of rounds specified by the user. Each block becomes equal to the hash of the previous block, the current block, and delta other blocks pseudorandomly chosen from the buffer based on the salt.
3. Extract: the last block of the buffer is output for key derivation.

Inputs:

- `key`: the key from the Balloon algorithm.
- `salt`: the salt provided to the Balloon algorithm.
- `spaceCost`: the space cost provided to the Balloon algorithm.
- `timeCost`: the time cost provided to the Balloon algorithm.
- `parallelism`: the parallelism provided to the Balloon algorithm.
- `length`: the length provided to the Balloon algorithm.
- `iteration`: the parallelism loop iteration from the Balloon algorithm.

Outputs:

- The last block of the buffer, which is `HASH_LEN` bytes long.

Steps:

~~~
buffer = BlockArray(spaceCost, HASH_LEN)
counter = 0

buffer[0] = PRF(key, LE64(counter++) || LE64(spaceCost) || LE64(timeCost) || LE64(parallelism) || LE64(length) || LE64(iteration))
for m = 1 to spaceCost - 1
    buffer[m] = PRF(key, LE64(counter++) || buffer[m - 1])

emptyKey = ByteArray(0)
previous = buffer[spaceCost - 1]
for t = 0 to timeCost - 1
    for m = 0 to spaceCost - 1
        pseudorandom = PRF(ZeroPad(emptyKey, HASH_LEN), LE64(counter++) || LE64(iteration) || salt)
        other1 = ReadLE64(pseudorandom.Slice(0, 8)) % spaceCost
        other2 = ReadLE64(pseudorandom.Slice(8, 8)) % spaceCost
        other3 = ReadLE64(pseudorandom.Slice(16, 8)) % spaceCost
        buffer[m] = PRF(key, LE64(counter++) || previous || buffer[m] || buffer[other1] || buffer[other2] || buffer[other3])
        previous = buffer[m]

return previous
~~~

# Implementation Considerations

There are several ways to optimise the pseudocode, which is written for readability:

- Instead of using an array of byte arrays for the buffer, access portions of a single large byte array.
- Instead of an integer counter that gets repeatedly converted to a byte array, allocate a byte array once and repeatedly fill that buffer or use a byte array counter.
- Instead of `x % spaceCost`, one can do `x & (spaceCost - 1)` because `spaceCost` is a power of 2.
- Skip the XORing of outputs when `parallelism = 1`.
- Create a single buffer full of zeros for the key derivation padding rather than padding the two variables separately.
- Instead of `Ceiling(length / HASH_LEN)`, one can do `(length + HASH_LEN - 1) / HASH_LEN`.
- Convert the key derivation domain separation string to bytes once rather than in each iteration of the loop.
- Use an incremental hash function API rather than manual concatenation.

# Choosing the Hash Function

The choice of cryptographic hash function affects the performance and security of Balloon in two ways:

1. For the same parameters, the attacker has an advantage if the algorithm is faster in hardware versus software. They will be able to do the computation in less time than the defender.
2. For the same delay, the defender will be forced to use smaller parameters with a slower cryptographic hash function in software. Using a faster algorithm in software means stronger parameters can be used.

It is RECOMMENDED to use a cryptographic hash function that is fast in software but relatively slow in hardware, such as BLAKE2b {{!RFC7693}}. As another example, SHA-512 is preferable to SHA-256 {{!RFC6234}}. Finally, SHA-3 {{FIPS202}} is NOT RECOMMENDED as it is slower in software compared to in hardware.

# Choosing the Parameters

The higher the `spaceCost` and `timeCost`, the longer it takes to compute an output. If these values are too small, security is unnecessarily reduced. If they are too large, there is a risk of user frustration and denial-of-service for different types of user devices and servers. To make matters even more complicated, these parameters may need to be increased over time as hardware gets faster/smaller.

The following procedure can be used to choose parameters:

1. For performing authentication on a server or running the algorithm on any type of user device, set the `parallelism` to 1. This avoids resource exhaustion attacks and slowdowns on machines with few CPU cores. Otherwise, set it to the maximum number of CPU cores the machine can dedicate to the computation.
2. Establish the maximum acceptable delay for the user. For example, 100-500 ms for authentication, 250-1000 ms for file encryption, and 1000-5000 ms for disk encryption. On servers, you also need to factor in the maximum number of authentication attempts per second.
3. Determine the maximum amount of memory available, taking into account different types of user devices and denial-of-service. For instance, mobile phones versus laptops/desktops.
4. Convert the MiB/GiB memory size that is a power of 2 to bytes. Then set `spaceCost` to `bytes / HASH_LEN`, which is the number of blocks.
5. Find the `timeCost` that brings you closest to the maximum acceptable delay or target number of authentication attempts per second by running benchmarks.
6. If `timeCost` is only 1, reduce `spaceCost` to be able to increase `timeCost`. Performing multiple rounds is beneficial for security {{AB17}}.

Regrettably, Balloon has not yet been sufficiently investigated for generic parameter recommendations to be made. This is also difficult given how various cryptographic hash functions can be used.

In all cases, it is RECOMMENDED to use a 128- or 256-bit `salt`. Other `salt` lengths SHOULD NOT be used, and the `salt` length SHOULD NOT vary in your protocol/application. See {{security-considerations}} for guidance on generating the `salt`.

For password hashing, it is RECOMMENDED to use a `length` of 128 or 256 bits. For key derivation, it is RECOMMENDED to use a `length` of at least 128 bits.

# Encoding Password Hashes

To store Balloon hashes in a database as strings, the following format SHOULD be used:

~~~
$balloon-hash$v=version$m=spaceCost,t=timeCost,p=parallelism$salt$hash
~~~

- `balloon-hash`: where `hash` is the official hash function OID minus any prefix (e.g. `id-`). For example, `blake2b512` for BLAKE2b-512 {{!RFC7693}}.
- `v=version`: this is version 2 of Balloon. If the design is modified, the version will be incremented.
- `m=spaceCost`: the memory size in blocks, not KiB.
- `t=timeCost`: the number of rounds.
- `p=parallelism`: the number of CPU cores/internal function calls in parallel.
- `salt`: the salt encoded in Base64 with no padding {{!RFC4648}}.
- `hash`: the full/untruncated Balloon output encoded in Base64 with no padding {{!RFC4648}}.

Here is an example encoded hash:

~~~
$balloon-sha256$v=1$m=1024,t=3,p=1$ZXhhbXBsZXNhbHQ$cWBD3/d3tEqnuI3LqxLAeKvs+snSicW1GVlnqmNEDfs
~~~

# Security Considerations

## Usage Guidelines

Technically, only preimage resistance is required for password hashing to prevent the attacker learning information about the password from the hash. However, non-collision-resistant hash functions (e.g. MD5 {{?RFC6151}} and SHA-1 {{?RFC6194}}) MUST NOT be used. Such functions are cryptographically weak and unsuitable for new protocols.

If possible, store the password in protected memory and/or erase the password from memory once it is no longer required. Otherwise, an attacker may be able to recover the password from memory or the disk.

The salt MUST be unique each time you call the function unless verifying a password hash or deriving the same key. It SHOULD be randomly generated using a cryptographically secure pseudorandom number generator (CSPRNG). However, it MAY be deterministic and predictable if random generation is not possible. It SHOULD be at least 128 bits long and SHOULD NOT exceed 256 bits.

The `spaceCost`, `timeCost`, and `parallelism` MUST be carefully chosen to avoid denial-of-service and user frustration whilst ensuring adequate protection against password cracking.

If you want to derive multiple keys (e.g. for encryption and authentication), you MUST only run the algorithm once and use different portions of the output as separate keys. Otherwise, the attacker may have an advantage, like only needing to run the algorithm once instead of twice to check a password, and you will be forced to use weaker parameters.

Avoid using hardcoded `spaceCost`/`timeCost`/`parallelism` parameters when performing password hashing; these SHOULD be stored as part of the password hash, as described in {{encoding-password-hashes}}. With key derivation, hardcoded parameters are acceptable if protocol versioning is used.

For password hashing, it is RECOMMENDED to encrypt password hashes using an authenticated encryption with associated data (AEAD) scheme {{?RFC5116}} before storage. This forces an attacker to compromise the key, which is stored separately from the database, as well as the database before they can begin password cracking. If the key is compromised but the database is not, it can be rotated without having to reset any passwords.

For key derivation, one can feed a secret key into the `pepper` parameter for additional security. This forces an attacker to compromise the pepper before they can guess the password. It is RECOMMENDED to use a 256-bit pepper.

## Security Guarantees

The security properties of Balloon depend on the chosen collision-resistant hash function. For example, a 256-bit hash typically provides 128-bit collision resistance and 256-bit (second) preimage resistance.

Balloon has been proven sequentially memory-hard in the random-oracle model and uses a password-independent memory access pattern to prevent side-channel attacks leaking information about the password {{BCS16}}. However, no function that uses a password-independent memory access pattern can be optimally memory-hard in the parallel setting {{AB16}}. In other words, Balloon is inherently weaker against parallel attacks.

To improve Balloon's resistance to parallel attacks, the output can be fed into a password hashing function with a password-dependent memory access pattern, such as scrypt {{?RFC7914}} or Argon2d {{?RFC9106}}. The cost of this approach is like increasing the `timeCost` of Balloon {{BCS16}}. However, even this does not defend against an attacker who can both a) obtain memory access pattern information and b) perform a massively parallel attack; it only protects against the two attacks separately.

Unlike password hashing algorithms such as bcrypt {{PM99}}, which perform many small pseudorandom reads, Balloon is not cache-hard. Whilst there are no known publications on cache-hardness at the time of writing, it is reported to provide better GPU/ASIC resistance than memory-hardness for shorter delays (e.g. < 1000 ms). In such cases, memory bandwidth and CPU cache sizes are bigger bottlenecks than total memory. This makes cache-hard algorithms ideal for authentication scenarios but potentially less suited for key derivation.

Third-party analysis of Balloon can be found in {{RD16}}, {{AB17}}, {{ABP17}}, and {{RD17}}. However, note that there are multiple versions of Balloon, and none of these papers have analysed the version specified in this document.

# IANA Considerations

This document has no IANA actions.

--- back

# Test Vectors

## Balloon-SHA-256

### Test Vector 1

~~~
password: 70617373776f7264

salt: 73616c74

spaceCost: 1

timeCost: 1

parallelism: 1

hash: 97a11df9382a788c781929831d409d3599e0b67ab452ef834718114efdcd1c6d
~~~

### Test Vector 2

~~~
password: 70617373776f7264

salt: 73616c74

spaceCost: 1

timeCost: 1

parallelism: 16

hash: a67b383bb88a282aef595d98697f90820adf64582a4b3627c76b7da3d8bae915
~~~

### Test Vector 3

~~~
password: 68756e7465723432

salt: 6578616d706c6573616c74

spaceCost: 1024

timeCost: 3

parallelism: 4

hash: 1832bd8e5cbeba1cb174a13838095e7e66508e9bf04c40178990adbc8ba9eb6f
~~~

### Test Vector 4

~~~
password:

salt: 73616c74

spaceCost: 3

timeCost: 3

parallelism: 2

hash: f8767fe04059cef67b4427cda99bf8bcdd983959dbd399a5e63ea04523716c23
~~~

# Acknowledgments
{:numbered="false"}

The original version of Balloon was designed by Dan Boneh, Henry Corrigan-Gibbs, and Stuart Schechter.

Thank you to Henry Corrigan-Gibbs and Steve Thomas for their help with the new design and feedback on this document.
