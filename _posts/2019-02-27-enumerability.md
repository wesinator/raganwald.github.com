---
title: "Enumerations, Denumerables, and Cardinals"
tags: [allonge,mermaid]
---

*Warning: This is an unfinished work. Feel free to share it on Twitter or other conventional social media, but I ask you not to post it on Hacker News or Reddit until it is finished.*

---

### enumerables and enumerations

In programming language jargon, an _enumerable_ is a value that can be accessed sequentially, or iterated over. Different languages use the term in slightly different ways, although they all have some relation to its basic definition.

In Mathematics, an [enumeration] is a complete, ordered list of all of the elements of a set or collection. There can be more than one enumeration for the same collection, and the collections need not have a finite number of elements.

[enumeration]: https://en.wikipedia.org/wiki/Enumeration

For example, here are two slightly different enumerations for the integers: `0, -1, 1, -2, 2, -3, 3, -4, 4, ...` and `0, -1, 1, 2, -2, -3, 3, 4, -4 ...`. We have no way to write out the complete enumeration, because although each integer is of finite size, there are an infinite number of them.

Naturally, not all enumerations are infinite in size. Here are two enumerations of the set ("whole numbers less than or equal to four"): `0, 1, 2, 3, 4` and `4, 3, 2, 1, 0`.

It is very interesting and useful that an enumeration is a separate entity from the collection it enumerates. We are going to work with enumerations in this essay.

---

### enumerations in javascript

In JavaScript, we can use generators as enumerations, e.g.

```javascript
function * integers () {
  yield 0;
  let i = 1;
  while (true) {
    yield -i;
    yield i++;
  }
}

integers()
  //=>
    0
    -1
    1
    -2
    2
    -3
    3
    -4
    4

    ...
```

In this essay, we will write the simplest possible enumerations, and build more complex enumerations using composition.

Sometimes, we want to parameterize a generator. Instead of writing a generator that takes parameters, we will consistently write functions that take parameters as arguments, and return simple generators that take no arguments.

So instead of writing:

```javascript
function * upTo (i, limit, by = 1) {
	for (let i = start; i <= limit; i += by) yield i;
}

function * downTo (i, limit, by = 1) {
	for (let i = start; i >= limit; i -= by) yield i;
}
```

We shall instead write:

```javascript
function upTo (start, limit, by = 1) {
  return function * upTo () {
    for (let i = start; i <= limit; i += by) yield i;
  };
}

function downTo (start, limit, by = 1) {
  return function * downTo () {
    for (let i = start; i >= limit; i -= by) yield i;
  }
}
```

These are functions that return generators, so we can write things like:

```javascript
const positives = upTo(1, Infinity);

for (const i of positives()) {
  console.log(i);
}
  //=>
    1
    2
    3
    4
    5

    ...
```

Just as functions can return generators, functions can also take generators as arguments and return new generators. Here's a function that merges a finite number of generators into a single enumeration:

```javascript
function merge (...generators) {
  return function * merged () {
    const iterables = generators.map(g => g());

    while (true) {
      const values =
        iterables
      	 .map(i => i.next())
         .filter(({done}) => !done)
         .map(({value}) => value)
      if (values.length === 0) break;
      yield * values;
    }
  }
}

const naturals = upTo(0, Infinity);
const negatives = downTo(-1, -Infinity);

const integers = merge(naturals, negatives);

for (const i of integers()) {
  console.log(i);
}
  //=>
    0
    -1
    1
    -2
    2
    -3
    3
    -4
    4

    ...
```

Thanks to composing simple parts, we wrote `const integers = merge(naturals, negatives)` instead of writing `function * integers () { ... }`. Here's another function that zips generators together. It has many uses, one of which is to put the output of a generator into a 1-to-1 correspondance with the natural numbers:

```javascript
function zip (...generators) {
  return function * zipped () {
    const iterables = generators.map(g => g());

    while (true) {
      const values =
        iterables
      	 .map(i => i.next())
         .filter(({done}) => !done)
         .map(({value}) => value)
      if (values.length === 0) break;
      yield values;
    }
  }
}

function * a () {
  let a = '';

  while (true) {
    yield a;
    a = `${a}a`;
  }
}

for (const s of zip(a, naturals)()) {
  console.log(s);
}
  //=>
    ["", 0]
    ["a", 1]
    ["aa", 2]
    ["aaa", 3]
    ["aaaa", 4]

    ...
```

We're going to make some more enumerations, and some tools for composing them, but before we do, let's talk about why  enumerations are interesting.

--

### denumerables and verification

*way too long. simply and possibly split in two after simplifying*

A [countable set] is any set (or collection) for which we can construct at least one enumeration. Or to put it in more formal terms, we can put the elements of the set into a one-to-one correspondance with some subset of the natural numbers. [Denumerables][countable set] are countable sets with an infinite number of elements, meaning they can be put into a 1-to-1 correspondance with the entire set of natural numbers.

[countable set]: https://en.wikipedia.org/wiki/Countable_set


When dealing with enumerating infinite sets, things can sometimes be tricky. If we say that an enumeration puts the elements of a denumerable into a 1-to-1 correspondance with the natural numbers, we must provide the following guarantee: **Every element of the enumeration correspondes to a finite natural number.** And in the case of an enumeration we write on a computer, it follows that for any member of the set, there is a natural number `n`, such that the member of the set appears as the `nth` element output by the enumeration.

So in our example of `function * a { ... }` above, we know that we can name any element, such as `aaaaaaaaaa`, and indeed, it will appear as the tenth output of the enumeration (and corresponds to the natural number `0`).

As we will see in more detail below, this sometimes means we must be careful how we arrange our enumerations. Recall `merge` from above.

```javascript
const integers = merge(naturals, negatives);

for (const i of integers()) {
  console.log(i);
}
  //=>
    0
    -1
    1
    -2
    2
    -3
    3
    -4
    4

    ...
```

Merge yields alternate elements from its constituent enumerations. But what if we had written this:

```javascript
function concatenate (...generators) {
  return function * concatenated () {
    for (const generator of generators) {
      yield * generator();
    }
  }
}

const notAllIntegers = concatenate(naturals, negatives);

notAllIntegers()
  //=>
    0
    1
    2
    3
    4
    5

    ...
```

As written, `concatenate` does **not** provide a correct enumeration over the integers, because we can name an integer, `-1`, that will not appear after a finite number of outputs. We can prove that, and the way we prove it helps get into the correct mindset for dealing with infinite enumerations:

Let's assume that `-1` does appear after a finite number of outputs. If we can show this leads to a contradiction, then we will know that our assumption is wrong, and that `-1` does not appear after a finite number of outputs.

We'll start by zipping `notAllIntegers` with the natural numbers:

```javascript
zip(naturals, notAllIntegers)()
  //=>
    [0, 0]
    [1, 1]
    [2, 2]
    [3, 3]
    [4, 4]
    [5, 5]

    ...
```

The proposition is that eventually, the enumeration outputs something like `[n, -1]`, where `n` is a natural number. What is `n`? As we can see, for any `n`, the actual output will be `[n, n]`, not `[n, -1]`. So concatenating the naturals with the negatives does **not** enumerate the integers.

Merging the naturals with the negatives does enumerate the integers, and we will use this idea repeatedly: When we want to enumerate the elements of multiple enumerations, we must find a way to interleave their elements.

---

### cardinality

The cardinality of a set is a measure of its size. Two sets have the same cardinality if their elements can be put into a one-to-one correspondence with each other. Cardinalities can also be compared. If the elements of set A can be put into a one-to-one correspondance with a subset of the elements of set B, but the elements of set B cannot be put into a one-to-one correspondence with set A, we say that A has a lower cardinality than B.

Obviously, the cardinalities of finite sets are natural numbers. For example, the cardinality of `[0, 1, 2, 3, Infinity]` is `4`, the same as its length.

The cardinality of infinite sets was studied by [Georg Cantor][Cantor]. As has been noted many times, the cardinality of infinite sets can be surprising to those thinking about it for the first time. For example, all infinite subsets of the natural numbers have the same cardinality as the natural numbers.

[Cantor]: https://en.m.wikipedia.org/wiki/Georg_Cantor

We can demonstrate that by putting them into a one-one-correspondance with the natural numbers:

```javascript
const naturals = upTo(0, Infinity);
const evens = upTo(0, Infinity, 2);

for (const s of zip(evens, naturals)()) {
  console.log(s);
}
  //=>
    [0, 0]
    [2, 1]
    [4, 2]
    [6, 3]
    [8, 4]

    ...
```

The even numbers have the same cardinality as the natural numbers. That is the paradox of infinity: The set of all even natural numbers is a proper subset of the set of all natural numbers, but nevertheless, they are the same size.

---

### products of enumerables

Sets can be created from the [cartesian product] (or simply "product") of two or more enumerables. For example, the set of all rational numbers is the product of the set of all natural numbers and the set of all positive natural numbers: A rational number can be expressed as a natural number numerator, divided by a positive natural number denominator.

The product of two sets can be visualized with a table. Here we are visualizing the rational numbers:

[Cartesian product]: https://en.wikipedia.org/wiki/Cartesian_product

|     |  1|  2|  3|...|
|-----|---|---|---|---|
|**0**|0/1|0/2|0/3|...|
|**1**|1/1|1/2|1/3|...|
|**2**|2/1|2/2|2/3|...|
|**3**|3/1|3/2|3/3|...|
|<strong>&vellip;</strong>|&vellip;|&vellip;|&vellip;| |<br/><br/>

There are plenty of naïve product functions. Here's one that operates on generators:

```javascript
function mapWith (fn, g) {
  return function * () {
    for (const e of g()) yield fn(e);
  }
}

function nprod2 (g1, g2) {
  return function * () {
  	for (const e1 of g1()) {
      console.log('1');
      for (const e2 of g2()) {
        yield [e1, e2];
      }
    }
  }
}

const zeroOneTwoThree = upTo(0, 3);
const oneTwoThree = upTo(1, 3);

const twelveRationals = mapWith(
  ([numerator, denominator]) => `${numerator}/${denominator}`,
  nprod2(zeroOneTwoThree, oneTwoThree)
);

for (const fraction of twelveRationals()) {
  console.log(fraction);
}
  //=>
    0/1
    0/2
    0/3
    1/1
    1/2
    1/3
    2/1
    2/2
    2/3
    3/1
    3/2
    3/3
```

The naïve approach iterates through all of the denominators members for each of the numerator's members. This is fast and simple, and works just fine for generators that only yield a finite number of elements. However, if we apply this to denumerables, it doesn't work:

```javascript
const naturals = upTo(0, Infinity);
const positives = upTo(1, Infinity);

const rationals =
      mapWith(
        ([numerator, denominator]) => `${numerator}/${denominator}`,
      	nprod2(naturals, positives)
	);

for (const s of rationals()) {
  console.log(s);
}
  //=>
    0/1
    0/2
    0/3
    0/4
    0/5
    0/6
    0/7
    0/8
    0/9
    0/10
    0/11
    0/12

    ...
```

A naïve product of two or more sets, where at least one of the sets is denumerable, is not an enumeration of the product of the sets. An enumeration of a denumerable guarantees that every element of the set appears in a finite number of outputs. Or likewise, that it puts the elements of the denumerable set into a one-to-one correspondance with the natural numbers.

The naïve product approach to enumerating the rationals does not output any element with a numerator greater than zero in a finite number of outputs. `14/6`, `19/62`, ... All of the fractions we can think of greater than zero never appear. They cannot be put into a one-to-one correspondance with the natural numbers.

This can be proven by assuming the contrary and then deriving a contradiction. Let us take a fraction greater than zero, `1962/614`. We are assuming that fractions with a non-zero numerator appear after a finite number of outputs, so there is some finite number, **n**, that represents the position of `1962/614` in the enumeration.

Let us scroll down the output looking for `n`. According to our algorithm, at position `n`, we will find `0/n`. But our assertion is that we will find `1962/614`. We can't find both, therefore the assumption that `1962/614` appears in a finite number of outputs is false, and the naïve product cannot enumerate denumerables.

---

### enumerating the product of denumerables

To enumerate the naïve product of two denumerables, we took the elements "row by row," to use the table visualization. This did not work, and neither would taking the elements column by column. What does work, is to take the elements _diagonal by diagonal_.

Here's our table again:

|     |  1|  2|  3|...|
|-----|---|---|---|---|
|**0**|0/1|0/2|0/3|...|
|**1**|1/1|1/2|1/3|...|
|**2**|2/1|2/2|2/3|...|
|**3**|3/1|3/2|3/3|...|
|<strong>&vellip;</strong>|&vellip;|&vellip;|&vellip;| |<br/><br/>

If we take the elements diagonal by diagonal, we will output: `0/1`, `0/2`, `1/1`, `0/3`, `1/2`, `2/1`, ...

Because of the order in which we access the elements of the generators, we cannot rely on iterating through the generators in order. We will build a random-access abstraction, `at()`. There is a simple schlemiel implementation:[^schlemiel]

[^schlemiel]: This is called a "schlemiel" implementation, because every time we wish to access a generator's element, we enumerate all of the elements of the generator from the beginning. This requires very little memory, but is wasteful of time. A memoized version is listed below.

```javascript
function at (generator, index) {
  let i = 0;

  for (const element of generator()) {
    if (i++ === index) return element;
  }
}

function * a () {
  let a = '';

  while (true) {
    yield a;
    a = `${a}a`;
  }
}

at(a, 42)
  //=> "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
```

With `at` in hand, our `prod2` function looks like this:

```javascript
function prod2 (g1, g2) {
  return function * () {
  	for (const sum of naturals()) {
      for (const [i1, i2] of zip(upTo(0, sum), downTo(sum, 0))()) {
        yield [at(g1, i1), at(g2, i2)];
      }
    }
  }
}

const rationals = mapWith(
  ([numerator, denominator]) => `${numerator}/${denominator}`,
  prod2(naturals, positives)
);

for (const rational of rationals()) {
  console.log(rational);
}
  //=>
    0/1
    0/2
    1/1
    0/3
    1/2
    2/1
    0/4
    1/3
    2/2
    3/1

    ...
```

As the product is output row-by-row, we can now say with certainty that no matter which element we care to name, it has appeared as the nth element for some finite value of n.

At the enormous cost of computing resources, we have an enumeration that enumerates the product of two denumerable sets, and we used it to enumeration rational numbers. This demonstrates that the rational numbers have the same cardinality of the natural numbers, and that any product of two denumerable sets is also denumerable.

---

### flattening denumerables

We previously looked at `merge`, a function that would merge a finite number of denumerables. It would not work for a denumerable number of denumerables, as it takes the elements "column-by-column," like a naïve product.

Consider this generator that generates exponents of natural numbers:

```javascript
const exp =
  n =>
    mapWith(
      p => Math.pow(n, p),
      upTo(1, Infinity)
    );

const twos = exp(2);

for (const powerOfTwo of twos()) {
  console.log(powerOfTwo);
}
  //=>
    2
    4
    8
    16
    32
    64
    128
    256
    512
    1024

    ...
```

If we compose it with a mapping, we can get a generator that generates generators that generate powers of natural numbers:

```javascript
const naturalPowers = mapWith(exp, naturals);

naturalPowers
  //=>
    0, 0, 0, 0, 0, ...
    1, 1, 1, 1, 1, ...
    2, 4, 8, 16, 32, ...
    3, 9, 27, 81, 243, ...
    4, 16, 64, 256, 1024, ...
    5, 25, 125, 625, 3125, ...
    6, 36, 216, 1296, 7776, ...
    7, 49, 343, 2401, 16807, ...

    ...
```

We now wish to merge all of these values into a single generator. We can use `prod2` to merge a denumerable number of denumerables with a little help from `at`.

We will call our function `flatten`:

```javascript
const flatten =
  denumerableOfDenumerables =>
    mapWith(
      ([denumerables, index]) => at(denumerables, index),
      prod2(denumerableOfDenumerables, naturals)
    );

flatten(naturalPowers)
  //=>
    0
    0
    1
    0
    1
    2
    0
    1
    4
    3

    ...
```

This verifies for us that the sum of a denumerable number of denumerables is also denumerable.

---

### exponentiation of denumerables

Back to products. The product of two denumerables is denumerable.

By inference, the product of three or more denumerables is also denumerable, because the denumerability of the product operation is transitive. Take denumerable sets `a`, `b`, `c`, and `d`. `a` and `b` are denumerable, and we know that `prod2(a, b)` is denumerable. Therefore `prod2(prod2(a, b), c)` is denumerable, and so is `prod2(prod2(prod2(a, b), c), d)`.

We can build `prod` on this basis. It's a function that takes a finite number of denumerables, and returns an enumeration over their elements, by using `prod2`:

```javascript
function prod (first, ...rest) {
  if (rest.length === 0) {
    return mapWith(
    	e => [e],
    	first
      );
  } else {
    return mapWith(
      ([eFirst, eRests]) => [eFirst].concat(eRests),
      prod2(first, prod(...rest))
    );
  }
}

const threeD = prod(naturals, naturals, naturals);

for (const triple of threeD()) {
  console.log(triple);
}
  //=>
    [0, 0, 0]
    [0, 0, 1]
    [1, 0, 0]
    [0, 1, 0]
    [1, 0, 1]
    [2, 0, 0]
    [0, 0, 2]
    [1, 1, 0]
    [2, 0, 1]
    [3, 0, 0]
    [0, 1, 1]
    [1, 0, 2]

    ...
```

We now have `prod`, a function that enumerates the product of any number of denumerables.

In arithmetic, exponentiation is the multiplying of a number by itself a certain number of times. For example, three to the power of 4 (`3^4`), is equivalent to three multiplied by three multiplied by three multiplied by three (`3*3*3*3`). Or we might say that it is the product of four threes.

We can take the exponent of denumerables as well. Here is the absolutely naïve implementation:

```javascript
function exponent (generator, n) {
  return prod(...new Array(n).fill(generator));
}

const threeD = exponent(naturals, 3);

for (const triple of threeD()) {
  console.log(triple);
}
  //=>
    [0, 0, 0]
    [0, 0, 1]
    [1, 0, 0]
    [0, 1, 0]
    [1, 0, 1]
    [2, 0, 0]
    [0, 0, 2]
    [1, 1, 0]
    [2, 0, 1]
    [3, 0, 0]
    [0, 1, 1]
    [1, 0, 2]

    ...
```

`exponent` works for any finite exponent of a denumerable set. When we were looking at flatten, we made a function, `exp`, that generates the exponents of a natural number.

We can do a similar thing with the exponents of a denumerable:

```javascript
const exponentsOf =
  generator =>
    mapWith(
      p => exponent(generator, p),
      upTo(1, Infinity)
    );

exponentsOf(naturals)
  //=>
    [0], [1], [2], [3], [4], [5], ...
    [0, 0], [0, 1], [1, 0], [0, 2], [1, 1], [2, 0], ...
    [0, 0, 0], [0, 0, 1], [1, 0, 0], [0, 1, 0], [1, 0, 1], [2, 0, 0], ...
    [0, 0, 0, 0], [0, 0, 0, 1], [1, 0, 0, 0], [0, 1, 0, 0], [1, 0, 0, 1], [2, 0, 0, 0], ...
    [0, 0, 0, 0, 0], [0, 0, 0, 0, 1], [1, 0, 0, 0, 0], [0, 1, 0, 0, 0], [1, 0, 0, 0, 1], [2, 0, 0, 0, 0], ...
    ...
````

It seems very extravagant to start thinking about an enumeration of enumerations of elements of a single denumerable (like the natural numbers), but we could look at all those elements another way: We are looking at all of the possible products of a denumerable.

We can flatten them into a single denumerable, of course:

```javascript
const products = generator => flatten(exponentsOf(generator));

products(naturals)
  //=>
    [0]
    [1]
    [0, 0]
    [2]
    [0, 1]
    [0, 0, 0]
    [3]
    [1, 0]
    [0, 0, 1]
    [0, 0, 0, 0]
    [4]
    [0, 2]
    [1, 0, 0]
    [0, 0, 0, 1]
    [0, 0, 0, 0, 0]
    [5]
    [1, 1]
    [0, 1, 0]
    [1, 0, 0, 0]
    [0, 0, 0, 0, 1]

    ...
```

It will take a good long while, but this generator will work its way diagonal by diagonal through all of the finite positive exponents of a denumerable, which shows that not only is the set of any one exponent of a denumerable denumerable, but so is the set of all exponents of a denumerable.

Of course, we are omitting one very important exponent, `0`. This only produces one product, `[]`. We will fix our `product` to include it:

```javascript
function cons (head, generator) {
  return function * () {
    yield head;
    yield * generator();
  }
}

const products = generator => cons([], flatten(exponentsOf(generator)));

products(naturals)()
  //=>
    []
    [0]
    [1]
    [0, 0]
    [2]
    [0, 1]

    ...
```

And now we have shown that the set containing all of the finite products (including what we might call the zeroth product) of a denumerable is also denumerable, by dint of having written an enumeration for it.

---

### the set of all finite subsets of a denumerable

The set of all finite products of a denumerable is interesting for several reasons. One of them is that the set of all finite products of a denumerable is a superset of the set of all finite subsets of a denumerable. Intuitively, it would sem that if we know that if we can enumerate the finite products of a denumerable, then the set of all finite subsets of a denumerable must also be enumerable.

The direct way to establish this is to write the enumeration we want. Let's start by establishing our requirement.

The set of all finite products of the natural numbers contains entries like `[]`, `[0]`, `[0, 0]`, `[0, 1]`, `[1, 0]`, `[0, 0, 0]`, `[0, 0, 1]`, `[0, 1, 0]`, `[1, 0, 0]`, `[0, 1, 1]`, &c. However for the purpose of enumerating the set of all finite subsets of a denumerable, the only sets that matter are `{}`, `{0}`, `{0, 1}`, .... The ordering of elements is irrelevant, as are duplicate elements.

We start with `combination`. Given a generator and a number of elements `k`, `combination` enumerates over all the ways that `k` elements can be selected from the generator's elements. Obviously, the k-combinations of a denumerable are also denumerable.

```javascript
function combination (generator, k) {
  if (k === 1) {
    return mapWith(
      e => [e],
      generator
    )
  } else {
    return flatten(
      mapWith(
        index => {
          const element = at(generator, index);
          const rest = slice(generator, index + 1);

          return mapWith(
            arr => (arr.unshift(element), arr),
            combination(rest, k - 1)
          );
        },
        naturals
      )
    );
  }
}

combination(naturals, 3)()
  //=>
    [0, 1, 2]
    [0, 1, 3]
    [1, 2, 3]
    [0, 2, 3]
    [1, 2, 4]
    [2, 3, 4]
    [0, 1, 4]
    [1, 3, 4]
    [2, 3, 5]
    [3, 4, 5]
    [0, 2, 4]
    [1, 2, 5]
    [2, 4, 5]
    [3, 4, 6]
    [4, 5, 6]
    [0, 3, 4]
    [1, 3, 5]
    [2, 3, 6]
    [3, 5, 6]
    [4, 5, 7]

    ...
```

Now what about all finite subsets? Well, that's very similar to `products`. We want a denumerable of denumerables, the first being all the subsets of size `1`, the next all the subsets of size `2`, then `3`, and so on. We flatten all those together, and cons the empty set onto the front:

```javascript
const subsets =
  generator =>
    cons(
      [],
      flatten(
        mapWith(
          k => combination(generator, k),
          naturals
        )
      )
    );

subsets(naturals)
  //=>
    []
    [0]
    [1]
    [0, 1]
    [2]
    [0, 2]
    [0, 1, 2]
    [3]
    [1, 2]
    [0, 1, 3]

    ...
```

And now we have shown that the set of all finite subsets of a denumerable, is also denumerable.

---

### taking the products of products

Back to `products`. To recap:

```javascript
const products = generator => cons([], flatten(exponentsOf(generator)));

products(naturals)()
  //=>
    []
    [0]
    [1]
    [0, 0]
    [2]
    [0, 1]
    [0, 0, 0]
    [3]
    [1, 0]
    [0, 0, 1]
    [0, 0, 0, 0]
    [4]
    [0, 2]
    [1, 0, 0]
    [0, 0, 0, 1]
    [0, 0, 0, 0, 0]
    [5]
    [1, 1]
    [0, 1, 0]
    [1, 0, 0, 0]
    [0, 0, 0, 0, 1]
    [0, 0, 0, 0, 0, 0]
    [6]
    [2, 0]
    [1, 0, 1]
    [0, 1, 0, 0]
    [1, 0, 0, 0, 0]
    [0, 0, 0, 0, 0, 1]
    [0, 0, 0, 0, 0, 0, 0]
    [7]
    [0, 3]
    [2, 0, 0]
    [1, 0, 0, 1]
    [0, 1, 0, 0, 0]
    [1, 0, 0, 0, 0, 0]
    [0, 0, 0, 0, 0, 0, 1]
    [0, 0, 0, 0, 0, 0, 0, 0]
    [8]
    [1, 2]
    [0, 0, 2]
    [2, 0, 0, 0]
    [1, 0, 0, 0, 1]
    [0, 1, 0, 0, 0, 0]
    [1, 0, 0, 0, 0, 0, 0]
    [0, 0, 0, 0, 0, 0, 0, 1]
    [0, 0, 0, 0, 0, 0, 0, 0, 0]

    ...
```

If we put this enumeration of `products(naturals)` into an explicit 1-to-1 correspondence with the natural numbers, we will see something interesting:

```
 0, []
 1, [0]
 2, [1]
 3, [0, 0]
 4, [2]
 5, [0, 1]
 6, [0, 0, 0]
 7, [3]
 8, [1, 0]
 9, [0, 0, 1]
10, [0, 0, 0, 0]
11, [4]
12, [0, 2]
13, [1, 0, 0]
14, [0, 0, 0, 1]
15, [0, 0, 0, 0, 0]
16, [5]
17, [1, 1]
18, [0, 1, 0]
19, [1, 0, 0, 0]

...
```

No number appearing within a product is greater than or equal to the ordinal of the product. For example, product `0` doesn't have any numbers. Product `1` contains a `0`. Product `2` contains a `1`. Product `3` contains two zeroes. And so forth. The products of an denumerable always outpace the elements of the denumerable.

Let's think of the numbers appearing within the products as indices. `0` is the first element of some denumerable, `1` is the second element, and so forth. We could substitute any denumerable for the natural numbers in `products(naturals)` by writing `products(someOtherDenumerable)`, but let's try an experiment:

What happens if we substitute the elements of the above output for the numbers... In itself?

Starting from the top, `[]` des not need any substitution, as it has no numbers:

```
 0, []
```

`[0]` becomes `[[]]`, because the `0th` element of the output is `[]`:

```
 0, []
 1, [[]]
```

Now `[1]` becomes `[[[]]]`:

```
 0, []
 1, [[]]
 2, [[[]]]
```

And `[0, 0]` becomes `[[], []]`:

```
 0, []
 1, [[]]
 2, [[[]]]
 3, [[], []]
```

Continuing this substitution from top to bottom, we get:

```
[]
[[]]
[[[]]]
[[], []]
[[[[]]]]
[[], [[]]]
[[], [], []]
[[[], []]]
[[[]], []]
[[], [], [[]]]
[[], [], [], []]
[[[[[]]]]]
[[], [[[]]]]
[[[]], [], []]
[[], [], [], [[]]]
[[], [], [], [], []]
[[[], [[]]]]
[[[]], [[]]]
[[], [[]], []]
[[[]], [], [], []]

...
```

Fortunately, we can create this enumeration without manually replacing elements or faffing about with `at` and `mapWith`, by writing a recursive generator:

```javascript
function * productsOfProducts () {
  yield * products(productsOfProducts)();
}

productsOfProducts()
  //=>
    []
    [[]]
    [[[]]]
    [[], []]
    [[[[]]]]

    ...
```

`productsOfProducts` is very interesting: If we think of each array as a node, and its children as arcs to other nodes, each element of this enumeration describes a finite tree, for example:

<div class="mermaid">
graph LR
  0(( ))
</div>

<div class="mermaid">
graph LR
  0(( ))-->1(( ))
</div>

<div class="mermaid">
graph LR
  0(( ))-->1(( ))
  1-->2(( ))
</div>

<div class="mermaid">
graph LR
  0(( ))-->1(( ))
  0-->2(( ))
</div>

<div class="mermaid">
graph LR
  0(( ))-->1(( ))
  1-->2(( ))
  2-->3(( ))
</div>

We have now established that `productsOfProducts` is an enumeration of every possible finite tree, and thus we know that the set of all finite trees is denumerable. It has the same cardinality as the set of all natural numbers.

---

### bonus

JavaScript happens to print arrays using `[` and `]` and `,`. But we could write our own "pretty printer" code. We could output the instructions for drawing graphs in SVG. Or we could output the arrays in a Lisp style:

```javascript
function prettyPrint(array) {
  function pp (array) {
    return `(${array.map(pp).join('')})`;
  }

  return array.map(pp).join('');
}

mapWith(pp, tree)()
  //=>
    ""
    "()"
    "(())"
    "()()"
    "((()))"
    "()(())"
    "()()()"
    "(()())"
    "(())()"
    "()()(())"
    "()()()()"
    "(((())))"
    "()((()))"
    "(())()()"
    "()()()(())"
    "()()()()()"
    "(()(()))"
    "(())(())"
    "()(())()"
    "(())()()()"

    ...
```

We are enumerating every finite [balanced parentheses][brutal] string.

[brutal]: http://raganwald.com/2019/02/14/i-love-programming-and-programmers.html "A Brutal Look at Balanced Parentheses, Computing Machines, and Pushdown Automata"