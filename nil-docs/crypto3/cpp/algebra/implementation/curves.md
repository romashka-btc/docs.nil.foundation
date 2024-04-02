# curves

## Elliptic Curves Architecture

Curves were built upon the `fields`. So it basically consists of several parts listed below:

1. Curve Policies
2. Curve g1, g2 group element arithmetic
3. Basic curve policies

![](<../../../../static/img/crypto3/image (3) (1).png>)

### Curve Policies

A curve policy describes its parameters, such as base field modulus `p`, scalar field modulus `q`, group element types `g1_type` and `g2_type`. It also contains `pairing_policy` type, needed for comfortable usage of curve pairing.

### Curve Element Algorithms

The curve element corresponds to a point of the curve and has all the needed methods and overloaded arithmetic operators. The corresponding algorithms are based on the underlying field algorithms that are also defined here.

### Basic Curve Policies

The main reason for the existence of a basic policy is that we need some of its parameters used in group elements and pairing arithmetic. So it contains such parameters that are needed by group element arithmetic, e.g. coeffs `a` and `b` or generator coordinates `x`, `y`. It also contains all the needed information about the underlying fields.