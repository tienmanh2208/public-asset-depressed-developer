# Find prime numbers in a range (Sieve of Eratosthenes method)

Suppose we want to find all the prime numbers between 2 and n. The brute force approach to this problem is to iterate through all the numbers in the range and check if that number is prime or not.

This approach is not optimal, we are not making use of the processed numbers. For example, if you know `k` is a prime number, so all the multiples of k in our range are not prime numbers. Using the pre-computed prime numbers to eliminate its multiples in the range is known as the Sieve of Eratosthenes.

## Sieve of Eratosthenes

You have a range from 1 to 20, now you want to find all the prime numbers in this range. The steps will be:

- We assume that all numbers in the range are prime numbers, mark them as potential candidates for prime numbers.
- Unmark `1` as the prime number, keep `2` as the prime number.
- You can see that `2` is the first prime number in the list, we will unmark all its multiples in the range as prime numbers. We will have the following result:

![Multiples of two](https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/algorithm/sive_of_eratosthenes/multiple_of_two.png)

- The next number is `3`, you can see that it's still marked as prime number, so we will unmark all its multiples in the range as prime numbers. The result would be:

![Multiples of three](https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/algorithm/sive_of_eratosthenes/multiple_of_three.png)

- The next number is `4`, this number is unmarked as prime number, so we will ignore this case.

- The next number is `5`, this number is marked as prime number, so we will unmark all its multiples as prime numbers.

- We will do the same logic for the rest of the list. In the end, we will have this result:

![Result](https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/algorithm/sive_of_eratosthenes/multiple_of_three.png)

## Explaination and sample code

### Explaination

Why we know for sure that number `3` is prime without checking it?. You can see that, a prime is a number that only be divisible by 1 and itself. Before `3`, we checked and unmark all prime numbers multiples to `FALSE`, so if `3` is divisible by any of the numbers less than 3, it had to be marked to `FALSE` because `3` will be multiple of one (or some) prime less than it, in case it's status still is `TRUE` so it's not divisible by any of the numbers less than `3`, so the only two factors of `3` is 1 and itself, that leds to `3` is a prime number.

We can generalize the above logic to number `k`. If `k` is not unmarked as prime number, it means `k` is not divisible by any number less than itself (Because if exist a number `x` less than `k` and `k` is divisible by `x` then `k` will be unmarked as prime number because `k` will be `x` multiple), so `k` will be prime number.

### Sample code

```py
def get_primes(n):
    # Initialize the array with n + 1 elements.
    # The index of each element will be the number we are checking.
    numbers = [True for x in range(n + 1)]

    # The first element is 0 and the second element is 1
    # we know for sure these numbers are not prime numbers.
    # So, we unmark them as prime numbers
    numbers[0] = False
    numbers[1] = False

    # We will go through the rest of the elements using logic of Eratosthenes.
    for i in range(2, n + 1):
        # If the current number is marked as False, it means this number is not prime numbers
        # We will ignore them
        if not numbers[i]:
            continue

        # The current number is prime number, so all its multiples are not prime numbers
        # So we mark them as False
        for j in range(i * 2, n + 1, i):
            numbers[j] = False

    # Get all primes from list
    primes = []

    for i in range(n + 1):
        if numbers[i]:
            primes.append(i)

    return len(primes)
```

## Optimize solution

The above solution is kind of fast, but we can optimize that. You can see that if we find a prime number, we will mark all its multiples to False. Assume we have `k` is the prime number, so we will mark all its multiples form `k * 2` to `k * j` (`k * j <= n` and `k * (j + 1) > n`). Actually, we just need to mark from `k * k` to `k * j` if j > k, otherwise we dont need to do anything at all.

Why is that? Because all the number from `k * 2` to `k * (k - 1)` is the multiples of numbers less than `k`, so no need to re-calculate them.

The code to mark the non-prime numbers would be

```py
# The current number is prime number, so all its multiples are not prime numbers
# So we mark them as False
for j in range(i * i, n + 1, i):
    numbers[j] = False
```

## Count (or find) the distinct prime factors of a number

If you want to find the distinct factors (or just counting distinct factors) of list of number, you can use this algorithm with a bit customization.

First, you will initialize the array with 0 is the default value (instead of True). And instead of marking the status of non-prime numbers to False, you just need to increase the value by 1. So, all the prime numbers go through the number will be recorded. **Note that** you can not use the optimal solution (that is only marking the number greater or equal with `k * k`) because we can loose some case.

If you want to get the distinct prime factors, instead of increasing the value by 1, you will initialize the array with default value is empty array, and you will push the prime number to the value if its multiple goes through the current index.