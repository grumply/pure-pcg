# pure-random-pcg

Implementation of the [RXS-M-XS](https://en.wikipedia.org/wiki/Permuted_congruential_generator#Variants) variant of the [permuted-congruential-generator](https://en.wikipedia.org/wiki/Permuted_congruential_generator) suite. RXS-M-XS was chosen for best browser performance.

## Features

`pure-random-pcg` comes with a convenient set of combinators.

### Generators 

Simple generators.

```haskell
newtype Generator a = Generator { generate :: Seed -> (Seed,a) }
```

A variate type-class for uniform bounded and unbounded random variate generation.

```haskell
class Variate a where
  uniform :: Generator a
  uniformR :: a -> a -> Generator a
```

### Applicative and Monadic generation

```haskell
data C = Double :+ Double

randomC :: Generator C
randomC = pure (:+) <*> uniform <*> uniform

randomC' :: Generator C
randomC' = do
  d1 <- uniformR 0 255
  d2 <- uniformR 0 255
  return (d1 :+ d2)
```

### Num instance

```haskell
inc :: Num a => Generator a -> Generator a
inc = (1 +)
```

### Reproducibility

`PCG` includes the ability to very quickly walk a `Seed` forwards or backwards via `advance` and `retract`.

```retract n . advance n == id```

```haskell
main = do
  seed0 <- newSeed
  let seed1 = advance 1000 seed0
      seed2 = retract 1000 seed1
  print (seed0 == seed2) -- True
```

```haskell
main = do
  seed0 <- newSeed
  let (seed1,_) = generate uniform seed0
      seed2 = retract 1 seed1
  print (seed0 == seed2) -- True
```

### Sampling

Sample from a distribution uniformly at random.

```haskell
names = sample ["Alice","Bob","Charlie","Dan","Elaine"]

main = do
  seed <- newSeed
  print (generate names seed)
```

Generate arbitrary bounded enumerables.

```haskell
data Dir = North | South | East | West deriving (Bounded,Enum,Show)

main = do
  seed <- newSeed
  let dirs :: [Dir]
      dirs = list choose seed
  print $ take 10 dirs
```

### Shuffling

Knuth Fisher-Yates shuffle.

```haskell
main = do
  seed <- newSeed
  print $ shuffle [1..10] seed
```

### Important Notes

On GHCJS and 32-bit GHC, this variant of pcg has a period of 2^32, meaning the pattern of random numbers will repeat after using the same `Seed` 2^32 times. If you need that many random values, use `independentSeed` to generate more seeds.

On 64-bit GHC, the period for this variant of pcg is 2^64, which you'd be unlikely to exhaust.

Keep in mind that pcg is **not** cryptographically secure.

### Performance

On a 2012 Intel i7-3770 @ 3.4GHz, this implementation achieves throughputs of +60Gb/s (64-bit ints); 1,000,000,000 per second.

On the same machine in Chrome v67, this implementation achieves thoughputs of 1Gb/s (32-bit ints); 33,000,000 per second.

RXS-M-XS has a much smaller period than MWC8222 or Mersenne Twister, but is 3-5x faster than SFMT(SIMD Fast Mersenne Twister), and 2-7x faster than MWC8222, and nearly 300-1500x faster than `random`'s System.Random.

In the end, you should pick the library with the API you like the best, as the performance of the RNG is dwarfed by what you do with the random numbers. Except for `System.Random`, don't use it when performance matters.

The implementation of SFMT doesn't naturally support bounded variate generation. All testing done using similar loops compiled with `-fllvm -O2`. SFMT was compiled with SSE2/SIMD enabled.

| Algorithm    | Int   | Int8  | Int16 | Int32 | Int64 |
| ------------ | ----- | ----- | ----- | ----- | ----- | 
| RXS-M-XS pcg | 1ns   | 1ns   | 1ns   | 1ns   | 1ns   | 
| SFMT         | 3.3ns | 3.3ns | 3.3ns | 3.3ns | 3.3ns |
| MWC8222      | 7ns   | 3.7ns | 3.7ns | 3.7ns | 7ns   |
| random       | 860ns | 300ns | 300ns | 527ns | 810ns |


| Algorithm    | Bounded Int  | Bounded Int8 | Bounded Int16 | Bounded Int32 | Bounded Int64  |
| ------------ | ------------ | ------------ | ------------- | ------------- | -------------- |
| RXS-M-XS pcg | 12ns/1ns*    | 1ns          | 1ns           | 1ns           | 5ns/1ns*       |
| MWC8222      | 22ns/7.4ns*  | 8.5ns        | 7.7ns         | 7.2ns         | 14.9ns/7.4ns*  |
| random       | 300-850ns**  | 300ns        | 300-310ns**   | 300-500ns**   | 300-850ns      |        

| Algorithm    | Word  | Word8 | Word16 | Word32 | Word64 |
| ------------ | ----- | ----- | ------ | ------ | ------ |
| RXS-M-XS pcg | 1ns   | 1ns   | 1ns    | 1ns    | 1ns    |
| SFMT         | 3.3ns | 3.3ns | 3.3ns  | 3.5ns  | 3.5ns  |
| MWC8222      | 7ns   | 3.7ns | 3.7ns  | 3.7ns  | 7ns    |
| random       | 800ns | 300ns | 300ns  | 500ns  | 810ns  |

| Algorithm    | Bounded Word  | Bounded Word8 | Bounded Word16 | Bounded Word32 | Bounded Word64  |
| ------------ | ------------- | ------------- | -------------- | -------------- | --------------- |
| RXS-M-XS     | 5.2ns/1ns*    | 1ns           | 1ns            | 1ns            | 5.2ns/1ns*      |
| MWC8222      | 17.3ns/7.8ns* | 8.5ns         | 7.7ns          | 7.2n           | 17.3ns/7.3ns*   |
| random       | 300-850ns**   | 300ns         | 300-310ns**    | 300-500ns**    | 300-850ns**     |

| Algorithm    | Double | Float | 
| ------------ | ------ | ----- | 
| RXS-M-XS pcg | 1ns    | 1ns   |
| SFMT         | 4.8ns  | -     |
| MWC8222      | 7.5ns  | 3.7ns |
| random       | 810ns  | 500ns |

| Algorithm    | Bounded Double | Bounded Float |
| ------------ | -------------- | ------------- |
| RXS-M-XS pcg | 1ns            | 1ns           |
| MWC8222      | 7.6ns          | 3.8ns         |
| random       | 1500ns         | 300-1000ns**  |

\* Bounded Ints/Int64/Word/Word64 in (lo,hi) can be more efficiently produced if (hi - lo < maxBound :: Int32)

\** random takes longer for larger ranges

### Thanks

This library was inspired by [Max Goldstein's](https://github.com/mgold) [elm-random-pcg](https://github.com/mgold/elm-random-pcg).

Much of the Distribution and range generation code comes from [Bryan O'Sullivan's](https://github.com/bos) [mwc-random](https://github.com/bos/mwc-random).