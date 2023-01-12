# Smallest multiple

## Problem

2520 is the smallest number that can be divided by each of the numbers from 1 to 10 without any remainder.

What is the smallest positive number that is evenly divisible(divisible with no remainder) by all of the numbers from 1 to N?

## Define solution

The first thought of mine is we will multiple all of them (from 1 to N), so we will get a number that can be divided by each of the numbers from 1 to N. But, It's not the smallest number.

Example: N = 4, if we multiple all numbers from 1 to 4, we get 24. But the smallest number that satisfied condition is 12. That because if that number can be divided by 4, it obviously can be divided by 2.

So, I come up with another idea, I will loop from N to 1 (let the current position is k), if k is divided by any smaller number, we will remove that number from the list (so we don't need to check that number again). In the end, we will have a list of numbers that don't have any 2 number are divisible by each other. And, I just need to multiple them and will get the result.

**But**, that solution have a bug. Assume n = 10, we will have the final array is [6, 7, 8, 9, 10]. If we multiple them, we will get 30240, and that's not the answer. Because we multiple 10 and 6 (=60) to get a number can be divided by them, but, actually, 30 is the smallest number, because 10 and 6 have the same common divider 2. So, My next idea would be, calculate the least common multiple (LCM) of the final set. It'll be like:

```js
lcm(a, b, c, d) = lcm(c, lcm(b,c,d)) = lcm(c, lcm(b, lcm(c,d)))
```

And we have the formular to calculate lcm:

$lcm(a,b)=\frac{|a.b|}{gcd(a,b)}$

gcd stands for greatest common divisor.

And finally, we will have the code for above solution

```js
function smallestMultiple(n) {
    let tempObj = {};

    for (let i = 1; i <= n; ++i) {
        tempObj[i] = 1;
    }

    for (let i = n; i > 1; --i) {
        if (!tempObj[i]) {
            continue;
        }

        for (let j = Math.floor(i / 2); j > 1; --j) {
            if (i % j == 0) {
                delete tempObj[j];
            }
        }
    }

    const allValues = Object.keys(tempObj);
    let result = 1;
    allValues.forEach((item) => {
        result = (result * item) / gcd(result, item);
    });

    console.log(result);
}

# gcd stands for greatest common divisor
function gcd(a, b) {
    if (b % a == 0) {
        return a
    }
    
    if (a % b == 0) {
        return b
    }
    
    let min = b < a ? b : a;
    let max = b > a ? b : a;
    
    let floor = Math.floor(min/2)
    for (let i = floor; i >= 1; --i) {
        if (a % i == 0 && b % i == 0) {
            return i;
        }
    }
}
```
