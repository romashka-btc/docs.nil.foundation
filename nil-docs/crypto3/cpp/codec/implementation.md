# implementation

Encoders/decoders usage is usually split into three stages:

1. Initialization. Implicit stage, in most cases no state scheduling is required.
2. Accumulation.
3. Encoding.
4. Finalization.

This separation defines the implementation architecture.

Some particular encoders merge the accumulation step with the encoding step. This means the input data block gets encoded as far as it is found filled with enough data. Others use "lazy" encoding, accumulating the unprocessed data before it gets encoded/decoded.

## Architecture Overview

Codec library architecture consists of several parts listed below:

1. Algorithms
2. Stream Processors
3. Codec Algorithms
4. Accumulators
5. Value Processors

## Algorithms

Implementation of a library is considered to be highly compliant with STL. So the crucial point is to have encodes to be usable in the same way as STL algorithms do.

STL algorithms library mostly consists of generic iterators and since C++20 range-based algorithms over generic concept-compliant types. Great example is `std::transform` algorithm:

```cpp
template<typename InputIterator, typename OutputIterator, typename UnaryOperation>
OutputIterator transform(InputIterator first, InputIterator last, OutputIterator out, UnaryOperation unary_op);
```

Input values of type `InputIterator` operate over any iterable range, no matter which particular type is supposed to be processed. While `OutputIterator` provides a type-independent output place for the algorithm to put results no matter which particular range this `OutputIterator` represents.

Since C++20 this algorithm got it analogous inside the Ranges library as follows:

```cpp
template<typename InputRange, typename OutputRange, typename UnaryOperation>
OutputRange transform(InputRange rng, OutputRange out, UnaryOperation unary_op);
```

This particular modification makes no difference if `InputRange` is a `Container` or something else. The algorithm is generic, just as data representation types are.

As much as such algorithms are implemented as generic ones, encoding algorithms should follow that too:

```cpp
template<typename BlockCipher, typename InputIterator, typename KeyIterator, typename OutputIterator>
OutputIterator encrypt(InputIterator first, InputIterator last, KeyIterator kfirst, KeyIterator klast, OutputIterator out);
```

\[`Codec`]\(@ref codec_concept) represents the particular block cipher that will be used. `InputIterator` represents the input data coming to be encrypted. Since block ciphers rely on a secret key `KeyIterator` represents the key data, and `OutputIterator` is exactly the same as it was in `std::transform` algorithm - it handles all the output storage operations.

The most obvious difference between `std::transform` is a representation of a policy defining the particular behaviour of an algorithm. `std::transform` proposes to pass it as a reference to `Functor`, which is also possible in the case of \[`Codec`]\(@ref codec_concept) policy used in function already pre-scheduled:

```cpp
template<typename BlockCipher, typename InputIterator, typename KeyIterator, typename OutputIterator>
OutputIterator encrypt(InputIterator first, InputIterator last, KeyIterator kfirst, KeyIterator klast, OutputIterator out);
```

Algorithms are no more than an internal structures initializer wrapper. In this particular case, the algorithm would initialize a stream processor fed with an accumulator set with \[`codec` accumulator]\(@ref accumulators::codec) inside initialized with \[`Codec`]\(@ref codec_concept) initialized with `KeyType` retrieved from input `KeyIterator` instances.

## Stream Data Processing

Encoders are usually defined for processing `Integral` value-typed byte sequences of specific size packed in blocks (e.g. `base64` is defined for byte sequences, which get processed per 4-byte block). Implementation supposes the input data to be a various-length input stream, which length could not be even to block size.

This requires an introduction of a stream processor specified with a particular parameter set unique for each \[`Codec`]\(@ref codec_concept) type, which takes input data stream and gets it split to blocks filled with converted to appropriate size integers (words in the cryptography meaning, not machine words).

Example. Let's assume the input data stream consists of 16 bytes as follows.

Let's assume the selected cipher to be used is Rijndael with a 32-bit word size, 128-bit block size and 128-bit key size. This means the input data stream needs to be converted to 32-bit words and merged into 128-bit blocks as follows:

Now with this a \[`Codec`]\(@ref block_cipher_concept) instance of \[`rijndael`]\(@ref block::rijndael) can be fed.

This mechanism is handled with `stream_processor` template class specified for each particular cipher with parameters required. Block ciphers suppose only one type of stream processor exists - the one which split the data into blocks, converts them, and passes them to `AccumulatorSet` reference as cipher input of format required. The rest of the data, not even to block size, gets converted too and fed value by value to the same `AccumulatorSet` reference.

## Data Type Conversion

Since block cipher algorithms are usually defined for `Integral` types or byte sequences of unique format for each cipher, encryption function being a generic requirement, should be handled with a particular cipher-specific input data format converter.

For example \[`rijndael` cipher]\(@ref block::rijndael) is defined over blocks of 32-bit words, which could be represented with `uint32_t`. This means all the input data should be in some way converted to a 4-byte-sized `Integral` type. In case of `InputIterator` is defined over some range of `Integral` value type, this is handled with plain byte repack, as shown in the previous section. This is a case with the input stream and required data format satisfying the same concept.

The more case with input data being presented by a sequence of various type `T` requires for the `T` to have a conversion operator `operator Integral()` to the type required by a particular \[`BlockCipher`]\(@ref block_cipher_concept) policy.

Example. Let us assume the following class is presented:

```cpp
class A {
public:
    std::size_t vals;
    std::uint16_t val16;
    std::char valc;
};
```

Now let us assume there exists an initialized and filled with random values `SequenceContainer` of value type `A`:

```cpp
std::vector<A> a;
```

To feed the \[`BlockCipher`]\(@ref block_cipher_concept) with the data presented, it is required to convert `A` to `Integral` type, which is only available if `A` has conversion operator in some way as follows:

```cpp
class A {
public:
    operator uint128_t() {
        return (vals << (3U * CHAR_BIT)) & (val16 << 16) & valc
    }

    std::size_t vals;
    std::uint16_t val16;
    std::char valc;
};
```

This part is handled internally with `stream_processor` configured for each particular cipher.

## Block Cipher Algorithms

Block cipher algorithms architecturally are stateful policies, which concepts regulate structural contents, and runtime content is scheduled key data. Block cipher policies are required to be compliant with \[`BlockCipher` concept]\(@ref block_cipher_concept).

\[`BlockCipher`]\(@ref block_cipher_concept) policies are required to be constructed with particular policy-compliant strictly-typed key data, usually represented by `BlockCipher::key_type`. This means constructing such a policy is quite a heavy task, so this should be handled carefully. The result of a \[`BlockCipher`]\(@ref block_cipher_concept) construction is filled and strictly-typed key schedule data member.

Once initialized with a particular key, \[`BlockCipher`]\(@ref block_cipher_concept) policy is not meant to be reinitialized but only destructed. Destruction of a \[`BlockCipher`]\(@ref block_cipher_concept) instance should zeroize key schedule data.

Usually, such a \[`BlockCipher`]\(@ref block_cipher_concept) policy would contain `constexpr static const std::size_t`-typed numerical cipher parameters, such as block bits, word bits, block words or cipher rounds.

Coming to typedefs contained in policy - they are meant to be mostly fixed-length arrays (usually `std::array`), which guarantees type safety and no occasional input data length issues.

Functions contained in policy are meant to process one block of strictly-typed data (usually it is represented by `block_type` typedef) per call. Such functions are stateful with respect to key schedule data represented by `key_schedule_type` and generated while block cipher constructor call.

## Accumulators

Encryption contains an accumulation step, which is implemented with [Boost.Accumulators](https://boost.org/libs/accumulators) library.

All the concepts are held.

Encoders and decoders contain pre-defined \[`accumulator_set`]\(@ref codec::accumulator_set), which is a `boost::accumulator_set` with pre-filled \[`codec` accumulator]\(@ref accumulators::codec).

Block accumulator accepts only one either `block_type::value_type` or `block_type` at insert.

Accumulator is implemented as a caching one. This means there is an input cache sized as same as particular `BlockCipher::block_type`, which accumulates unprocessed data. After it gets filled, data gets encrypted; it gets moved to the main accumulator storage and the cache empties.

\[`block` accumulator]\(@ref accumulators::block) internally uses \[`bit_count` accumulator]\(@ref accumulators::bit_count) and designed to be combined with other accumulators available for [Boost.Accumulators](https://boost.org/libs/accumulators).

Example. Let's assume there is an accumulator set whose intention is to encrypt all the incoming data with \[`rijndael<128, 128>` cipher]\(@ref block::rijndael) and to compute a \[`sha2<256>` hash]\(@ref hashes::sha2) of all the incoming data as well.

This means there will be an accumulator set defined as follows:

```cpp
using namespace boost::accumulators;
using namespace nil::crypto3;

boost::accumulator_set<
    accumulators::block<block::rijndael<128, 128>>,
    accumulators::hash<hashes::sha2<256>>> acc;
```

Extraction is supposed to be defined as follows:

```cpp
std::string hash = extract::hash<hashes::sha2<256>>(acc);
std::string ciphertext = extract::block<block::rijndael<128, 128>>(acc);
```

## Value Postprocessors

Since the accumulator output type is strictly tied to \[`digest_type`]\(@ref block::digest_type) of particular \[`BlockCipher`]\(@ref block_cipher_concept) policy, the output format in generic is closely tied to the digest type too. Digest type is usually defined as a fixed or variable length byte array, which is not always the format of the container or range user likes to store the output in. It could easily be a `std::vector<uint32_t>` or a `std::string`, so there is a \[`cipher_value`]\(@ref cipher_value) state holder, which is made to be implicitly convertible to various container and range types with internal data repacking implemented.

Such a state holder is split into a couple of types:

1. Value holder. Intended to have an internal output data storage. Actually stores the `AccumulatorSet` digest data.
2. Reference holder. Intended to store a reference to external `AccumulatorSet`, which is usable in case of data gets appended to the existing accumulator.