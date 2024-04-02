# pack

## Introduction

This document provides a detailed description of pack algorithms used in the current project. Briefly, pack algorithms are used throughout the project to transform data divided into chunks with one parameter into data divided into chunks with another parameter. The dependence of algorithms on these parameters is discussed further in the Section [Algorithms](##pack_algorithms).

## Basic notions and assumptions

This section introduces notions we will use throughout the document.

We will assume further that the data to be transformed by pack algorithms is byte-aligned. In addition, the packed data is also considered to be byte-aligned.

We will also suppose that the data (both input and output) is divided into chunks which are the groups of bytes for which the type is language-defined (such as `uint8_t` or `uint32_t`) or user-defined. Currently, we restrict the chunk type to integral.

By term _endianness,_ we mean the significance order of groups of bytes, further called _units,_ combined with the significance order of bits inside of each unit. For example, bit*unit_little_bit endianness refers to the \_most significant unit first* order, and each unit contains the bits in the _least significant bit first_ order.\
Generally, we have the four following types of endianness:

1. `big_unit_big_bit` endianness refers to the _most significant unit in the first_ order, and each unit contains the bits in the `most significant bit first` order;
2. `little_unit_big_bit` endianness refers to the _least significant unit in the first_ order, and each unit contains the bits in the `most significant bit first` order;
3. `big_unit_little_bit` endianness refers to the _most significant unit in the first_ order, and each unit contains the bits in the `least significant bit first` order;
4. `little_unit_little_bit` endianness refers to the _least significant unit in the first_ order, and each unit contains the bits in the `least significant bit first` order.

Note that if the unit is the byte, the first two endianness coincide with well-known big-endian and little-endian byte orders. However, we prefer to use the above-introduced classification since it takes into account the architectures with non-typical endianness and, hence, is wider than just the big/little-endian dichotomy.

All the notations not described in this section will be introduced on the fly.

## Algorithms

Pack algorithms are intended to transform byte-aligned data divided into chunks of bit size denoted by `InputValueBits` into byte-aligned data divided into chunks of bit size denoted by `OutputValueBits`. Moreover, we suppose that all input and output data chunks consist of units ordered in accordance with the corresponding endianness. We will refer to this endianness as `InputEndianness` and `OutputEndianness`, respectively.

Pack algorithms are divided into the following categories depending on the relation between the chunk sizes:

- `InputValueBits` < `OutputValueBits`. This algorithm combines several small chunks into a big one and is further referred to as an imploder. It is also supposed that `OutputValueBits` is a multiple of `InputValueBits`.
- `InputValueBits` > `OutputValueBits`. This algorithm splits one big chunk into several small chunks, and is further referred to as an exploder. It is also supposed that `InputValueBits` is a multiple of `OutputValueBits`.
- `InputValueBits` = `OutputValueBits`. This algorithm transforms data chunk-by-chunk in accordance with the corresponding endianness conversion.

It is important to note that the combining and splitting operations in imploder and exploder algorithms are also dependent on endianness conversion.

### Endianness conversion

Consider first the case of `little_unit_big_bit`-to-`big_unit_big_bit` conversion.

```cpp
std::array<uint16_t, 2> input = {0x1234, 0x5678};
std::array<uint32_t, 1> output = {0x34127856};
```

In the example above, we suppose that the unit of arrays is a byte. It is easy to see that the value written to the out array is obtained by combining each input chunk byte data in reverse byte order.

It may seem at first look that all same endianness conversions are simplicity itself, but that's not quite true. To dive deeper into the problem of endianness conversion, consider the inverse conversion.

```cpp
std::array<uint16_t, 4> input = {0x1234, 0x5678};
std::array<uint32_t, 2> output {0x78563412};
```

output:

In this example, `input` array units are ordered in `big_unit_big_bit` endianness and `output` array units are ordered in `little_unit_big_bit` endianness (supposing that the unit is a byte). In addition, we have the reverse order of input chunks in the out array to reverse byte order.

An interested reader may wonder why changing of endianness leads to such a strange effect. Well, the answer to this question lies in the following convention: all data is divided into chunks with units ordered in `big_unit_big_bit` endianness will stay unchanged when transforming to data with chunk units ordered in `big_unit_big_bit` endianness. Let us explain it with the following example.

```cpp
std::array<uint16_t, 4> input = {0x1234, 0x5678, 0x90ab, 0xcdef};
std::array<uint64_t, 1> output = {0x1234567890abcdef};
```

Here it is easy to see that the data `input` was just concatenated into the `output` data with no additional transformations. Now, the first and second examples described in this section implicitly rely on the above-described convention. In the first example, the input data is concatenated in reverse byte order, and in the second example, the byte order is reversed after the input data concatenation.

We haven't touched the case of endian conversion with bit reversals yet. Let us see at the following example:

```cpp
std::array<uint8_t, 4> input = {0x12, 0x34, 0x56, 0x78};
std::array<uint16_t, 2> output = {0x482c, 0x6a1e};
```

In this example, `input` array units are ordered in `big_unit_little_bit` endianness and `output` array units are ordered in `big_unit_big_bit` endianness (supposing that the unit is byte). Writing the byte `0x12` in binary form gives us `00010010`, its reverse binary form is `01001000`, which gives us `0x48` in hex representation. The same transformations are applied to the remaining bytes.

To conclude, there are three types of reversals that we must deal with in pack algorithms:

1. data chunk order reversal (as in `big_unit_big_bit`-to-`little_unit_big_bit` conversion);
2. unit order reversal (as in `little_unit_big_bit`-to-`big_unit_big_bit` conversion);
3. bit order reversal (as in `big_unit_little_bit`-to-`big_unit_big_bit` conversion).

### Data chunk order reversal

In this section, we suppose the chunk type of input and output data is integral.

Data chunk order reversal transforms a group of consecutive input chunks with units in `InputEndianness` order into an output chunk with units in `OutputEndianness` order and can be described as follows.

1. Check whether `InputEndianness` or `OutputEndianness` is `little_bit`. This condition determines the data chunk order reversal presence or absence. (We have already seen how the order of chunks changed in `big_unit_big_bit` -to-`little_unit_big_bit` conversion, so this is just a generalization.)\
   The choice of endianness depends on an algorithm where this step is carried out (imploder or exploder).
2. If the endianness on the previous step is, set shift equal to `OutputBits` - (`InputBits` + already_processed_bits) in the case of imploder, and to `InputBits` - (`OutputBits` + already_processed_bits) in the case of the exploder. Otherwise, set shift equal to already_processed_bits.

By already_processed_bits, we mean the number of already processed (i.e. combined or split) input chunks multiplied by the number of bits in a byte. The shift is later used either to retrieve or to accumulate input chunks ( see \[`Imploder`]\(@ref pack_imploder)).

### Unit order reversal

This section considers that the unit is no less than a byte.

Unit order reversal transforms the order of units in each input chunk. It can be described as follows:

1. If `InputEndianness` and `OutputEndianness` have different unit orders, reverse byte order in the input chunk and go to the next step. Otherwise, do nothing and return.
2. Reverse byte order in each input chunk unit.

Note that the second step can be omitted if the unit is a byte.

### Bit order reversal

This section considers that the unit is no less than a byte.

Bit order reversal transforms the order of bits in each input chunk unit. It can be described as follows:

1. If `InputEndianness` and `OutputEndianness` have different bit orders, reverse byte order in each input chunk unit and go to the next step. Otherwise, do nothing and return.
2. Reverse bit order in each byte of input chunk unit.

Note that the first step can be omitted if the unit is a byte.

### Imploder

Recall that imploder algorithm deals with the case `InputValueBits` < `OutputValueBits` and converts data from `InputEndianness` to `OutputEndianness` order.

There are three main parts of imploder algorithm:

1. Calculation of the value that indicates the position of the input chunk in the output and indicates data chunk order reversal, if present. This part is currently implemented via shift trait containing the value that depends on whether the output endianness is `little_unit`.
2. Unit order reversal algorithm. This part is implemented via partial struct specializations, which deal with different specific cases.
3. Bit order reversal algorithm. This part is implemented via partial struct specializations, which deal with different specific cases.

The described process can be written in the following pseudocode:

```cpp
input_chunk = first input chunk

for each output_chunk:
	output_chunk = 0
	already_processed_bits = 0

	while already_processed_bits != OutputValueBits:
		/* Step 1 */
		if OutputEndianness is little_unit:
			shift = already_processed_bits
		else
			shift = OutputValueBits - (InputValueBits + already_processed_bits)

		/* Step 2 */
		tmp = input_chunk
		if InputEndianness and OutputEndianness have not same unit order:
			reverse_unit_order(tmp)

		/* Step 3 */
		if InputEndianness and OutputEndianness have not same bit order:
			reverse_bit_order(tmp)

		/* Data concatenation */
		output_chunk = output_chunk OR (tmp << shift)
		already_processed_bits = already_processed_bits + InputValueBits

		input_chunk = next input chunk
```

Here `OR` denotes logical OR operation and `<<` denotes left shift operation.

### Exploder

The Exploder algorithm deals with the case `InputValueBits` > `OutputValueBits`, converts data from `InputEndianness` to `OutputEndianness` order and is the same as the imploder algorithm described in Section \[Imploder]\(@ref pack_imploder) except for several points:

- the condition of shift choice is replaced with `InputEndianness` instead of `OutputEndianness`;
- right shift operation instead of a left shift operation is used;
- the output chunk is just part of the input chunk.

The pseudocode of the exploder with the above-described changes is presented below.

```cpp
take first output_chunk

for each input_chunk:
	already_processed_bits = 0

	while already_processed_bits != InputValueBits:
		/* Step 1 */
		if InputEndianness is little_unit:
			shift = already_processed_bits
		else
			shift = InputValueBits - (OutputValueBits + already_processed_bits)

		/* Splitting input data */
		tmp = input_chunk >> shift

		/* Step 2 */
		if InputEndianness and OutputEndianness have not same unit order:
			reverse_unit_order(tmp)

		/* Step 3 */
		if InputEndianness and OutputEndianness have not same bit order:
			reverse_bit_order(tmp)

		/* Writing result of transformations to output_chunk */
		output_chunk = tmp
		already_processed_bits = already_processed_bits + OutputValueBits

		take next output_chunk
```

### Equal size case

This algorithm deals with the case `InputValueBits` = `OutputValueBits` and converts data from `InputEndianness` to `OutputEndianness`. It uses the aforementioned unit, and bit order reversal algorithms, and its pseudocode can be presented as follows:

```cpp
take first output_chunk

for each input_chunk:
	tmp = input_chunk

	if InputEndianness and OutputEndianness have not same unit order:
		reverse_unit_order(tmp)

	if InputEndianness and OutputEndianness have not same bit order:
		reverse_bit_order(tmp)

	output_chunk = tmp

	take next output_chunk
```