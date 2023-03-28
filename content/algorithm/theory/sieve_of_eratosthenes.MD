# Find prime numbers in a range (Sieve of Eratosthenes method)

Suppose we want to find all the prime numbers between 2 and n. The brute force approach to this problem is to iterate through all the numbers in the range and check if that number is prime or not.

This approach is not optimal, we are not making use of the processed numbers. For example, if you know `k` is a prime number, so all the multiples of k in our range are not prime numbers. Using the pre-computed prime numbers to eliminate its multiples in the range is known as the Sieve of Eratosthenes.

## Sieve of Eratosthenes

You have a range from 1 to 20, now you want to find all the prime numbers in this range. The steps will be:

- We assume that all numbers in the range are prime numbers, mark them as potential candidates for prime numbers.
- Unmark `1` as the prime number, keep `2` as the prime number.
- You can see that `2` is the first prime number in the list, we will unmark all its multiples in the range as prime numbers. We will have the following result:

// TODO: images

- The next number is `3`, you can see that it's still marked as prime number, so we will unmark all its multiples in the range as prime numbers. The result would be:

// TODO: images

- The next number is `4`, this number is unmarked as prime number, so we will ignore this case.

- The next number is `5`, this number is marked as prime number, so we will unmark all its multiples as prime numbers.

- We will do the same logic for the rest of the list. In the end, we will have this result:

// TODO: images

## Explaination and sample code

Why we know for sure that number `3` is prime without checking it?. You can see that, a prime is a number that only be divisible by 1 and itself. Before `3`, we checked and unmark all prime numbers multiples to `FALSE`, so if `3` is divisible by any of the numbers less than 3, it had to be marked to `FALSE` because `3` will be multiple of one (or some) prime less than it, in case it's status still is `TRUE` so it's not divisible by any of the numbers less than `3`, so the only two factors of `3` is 1 and itself, that leds to `3` is a prime number.

We can generalize the above logic to number `k`. If `k` is not unmarked as prime number, it means `k` is not divisible by any number less than itself (Because if exist a number `x` less than `k` and `k` is divisible by `x` then `k` will be unmarked as prime number because `k` will be `x` multiple), so `k` will be prime number.
