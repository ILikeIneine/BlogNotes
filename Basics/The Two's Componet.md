# Two's Complement

Straight to the hell:

## Why Inversion and Adding One Works

Invert and add one, Invert and add one. It works, but why?

Inverting and adding one might sound like a stupid thing to do, but it's actually just a mathematical shortcut of a rather straightforward computation.

### Borrowing and Subtraction

Remember the old trick we learned in first grade of "borrowing one's" from future ten's places to perform a subtraction? You may not, so I'll go over it. As an example, I'll do 93702 minus 58358.

```txt
  93702
- 58358
-------
```

Now, then, what's the answer to this computation? We'll start at the least significant digit, and subtract term by term. We can't subtract 8 from 2, so we'll borrow a digit from the next most significant place (the tens place) to make it 12 minus 8. 12 minus 8 is 4, and we note a 1 digit above the ten's column to signify that we must remember to subtract by one on the next iteration.

```txt
     1
  93702
- 58358
-------
      4
```

This next iteration is 0 minus 5, and minus 1, or 0 minus 6. Again, we can't do 0 minus 6, so we borrow from the next most significant figure once more to make that 10 minus 6, which is 4.

```txt
    11
  93702
- 58358
-------
     44
```

This next iteration is 7 minus 3, and minus 1, or 7 minus 4. This is 3. We don't have to borrow this time.

```txt
    11
  93702
- 58358
-------
    344
```

This next iteration is 3 minus 8. Again, we must borrow to make thi 13 minus 8, or 5.

```txt
  1 11
  93702
- 58358
-------
   5344
```

This next iteration is 9 minus 5, and minus 1, or 9 minus 6. This is 3. We don't have to borrow this time.

```txt
  1 11
  93702
- 58358
-------
  35344
```

So 93702 minus 58358 is 35344.

### Borrowing and it's Relevance to the Negative of a Number

When you want to find the negative of a number, you take the number, and subtract it from zero. Now, suppose we're really stupid, like a computer, and instead of simply writing a negative sign in front of a number A when we subtract A from 0, we actually go through the steps of subtracting A from 0.

Take the following idiotic computation of 0 minus 3:

```txt
              1        11       111      1111
000000    000000    000000    000000    000000
-    3    -    3    -    3    -    3    -    3
------    ------    ------    ------    ------
               7        97       997      9997
```

Et cetera, et cetera. We'd wind up with a number composed of a 7 in the one's digit, a 9 in every digit more significant than the 100's place.

### The Same in Binary

We can do more or less the same thing with binary. In this example I use 8 bit binary numbers, but the principle is the same for both 8 bit binary numbers (chars) and 32 bit binary numbers (ints). I take the number 75 (in 8 bit binary that is 010010112) and subtract that from zero.

Sometimes I am in the position where I am subtracting 1 from zero, and also subtracting another borrowed 1 against it.

```txt
                      1            11           111          1111
  00000000      00000000      00000000      00000000      00000000
- 01001011    - 01001011    - 01001011    - 01001011    - 01001011
----------    ----------    ----------    ----------    ----------
                       1            01           101          0101

    11111        111111       1111111      11111111
  00000000      00000000      00000000      00000000
- 01001011    - 01001011    - 01001011    - 01001011
----------    ----------    ----------    ----------
     10101        110101       0110101      10110101
```

If we wanted we could go further, but there would be no point. Inside of a computer the result of this computation would be assigned to an eight bit variable, so any bits beyond the eighth would be discarded.

With the fact that we'll simply disregard any extra digits in mind, what difference would it make to the end result to have subtracted 01001011 from 100000000 (a one bit followed by 8 zero bits) rather than 0? There is none. If we do that, we wind up with the same result:

```txt
 11111111
 100000000
- 01001011
----------
 010110101
```

So to find the negative of an n-bit number in a computer, subtract the number from 0 or subtract it from 2n. In binary, this power of two will be a one bit followed by n zero bits.

In the case of 8-bit numbers, it will answer just as well if we subtract our number from (1 + 11111111) rather than 100000000.

```txt
         1
+ 11111111
- 01001011
----------
```

In binary, when we subtract a number A from a number of all 1 bits, what we're doing is inverting the bits of A. So the subtract operation is the equivalent of inverting the bits of the number. Then, we add one.

So, to the computer, taking the negative of a number, that is, subtracting a number from 0, is the same as inverting the bits and adding one, which is where the trick comes from.

## End

Love from [2000's](https://www.cs.cornell.edu/~tomf/notes/cps104/twoscomp.html).
