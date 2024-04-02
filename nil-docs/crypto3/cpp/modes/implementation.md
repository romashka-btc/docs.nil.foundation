# implementation

Cipher modes usage is usually split into three stages:

1. Initialization. Implicit stage with the creation of accumulator to be used.
2. Accumulation. Performed one or more times. Calling an update several times is equivalent to calling it once with all of the arguments concatenated.
3. Finalization. Accumulated hash data is required to be finalized, padded and prepared to be retrieved by the user.

## Architecture Overview

Hashes library architecture consists of several parts listed below:

1. Algorithms
2. Stream Processors
3. Constructions and Compressors
4. Accumulators
5. Value Processors

## Algorithms

Implementation of a library is considered to be highly compliant with STL. So the crucial point is to have ciphers to be usable in the same way as STL algorithms do.

STL algorithms library mostly consists of generic iterators and since C++20 range-based algorithms over generic concept-compliant types. Great example is `std::transform` algorithm:

```cpp
template<typename InputIterator, typename OutputIterator, typename UnaryOperation>
OutputIterator transform(InputIterator first, InputIterator last, OutputIterator out, UnaryOperation unary_op);
```

Input values of type `InputIterator` operate over any iterable range, no matter which particular type is supposed to be processed. While `OutputIterator` provides a type-independent output place for the algorithm to put results no matter which particular range this `OutputIterator` represents.

Since C++20, this algorithm got it analogous inside the Ranges library as follows:

```cpp
template<typename InputRange, typename OutputRange, typename UnaryOperation>
OutputRange transform(InputRange rng, OutputRange out, UnaryOperation unary_op);
```

This particular modification makes no difference if `InputRange` is a `Container` or something else. The algorithm is generic, just as data representation types are.

As much as such algorithms are implemented as generic ones, hash algorithms should follow that too:

```cpp
template<typename Hash, typename InputIterator, typename OutputIterator>
OutputIterator hash(InputIterator first, InputIterator last, OutputIterator out);
```

`Hash` is a policy type which represents the particular hash that will be used. `InputIterator` represents the input data coming to be hashed. `OutputIterator` is exactly the same as it was in `std::transform` the algorithm - it handles all the output storage operations.

The most obvious difference between `std::transform` is a representation of a policy defining the particular behaviour of an algorithm. `std::transform` proposes to pass it as a reference to `Functor`, which is also possible in case of `Hash` policy used in function already pre-scheduled:

```cpp
template<typename Hash, typename InputIterator, typename OutputIterator>
OutputIterator hash(InputIterator first, InputIterator last, OutputIterator out);
```

Algorithms are no more than an internal structures initializer wrapper. In this particular case, the algorithm would initialize a stream processor fed with an accumulator set with \[`hash` accumulator]\(@ref accumulators::hash) inside initialized with `Hash`.

## Stream Data Processing

Hashes are usually defined for processing `Integral` value typed byte sequences of specific size packed in blocks (e.g. \[sha2]\(@ref nil::crypto3::hashes::sha2) is defined for blocks of words which are actually plain `n`-sized arrays of `uint32_t` ). Input data in the implementation proposed is supposed to be a various-length input stream, which length could not be even to block size.

This requires the introduction of a stream processor specified with a particular parameter set unique for each \[`Hash`]\(@ref modes_concept) type, which takes input data stream and gets it split to blocks filled with converted to appropriate size integers (words in the cryptography meaning, not machine words).

Example. Let's assume the input data stream consists of 16 bytes as follows.

Now with this a \[`Hash`]\(@ref modes_concept) instance of \[sha2]\(@ref nil::crypto3::hashes::sha2) can be fed.

This mechanism is handled with `stream_processor` template class specified for each particular cipher with parameters required. Hashes suppose only one type of stream processor exist - the one which split the data into blocks converts them and passes them to `AccumulatorSet` reference as cipher input of the format required. The rest of the data, not even to block size, gets converted too and fed value by value to the same `AccumulatorSet` reference.

## Data Type Conversion

Since block cipher algorithms are usually defined for `Integral` types or byte sequences of unique format for each cipher, encryption function being a generic requirement, should be handled with a particular cypher-specific input data format converter.

For example `Rijndael` cipher is defined over blocks of 32-bit words, which could be represented with `uint32_t`. This means all the input data should be in some way converted to a 4-byte sized `Integral` type. In case of `InputIterator` is defined over some range of `Integral` value type, this is handled with plain byte repack, as shown in the previous section. This is a case with the input stream and required data format satisfying the same concept.

The more case with input data being presented by a sequence of various type `T` requires for the `T` to have conversion operator `operator Integral()` to the type required by particular `BlockCipher` policy.

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

To feed the `BlockCipher` with the data presented, it is required to convert `A` to `Integral` type, which is only available if `A` has conversion operator in some way as follows:

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

## Hash Algorithms

## Accumulators

## Value Postprocessors