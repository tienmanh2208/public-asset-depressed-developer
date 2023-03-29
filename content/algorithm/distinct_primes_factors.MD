# Distinct primes factors

Ref: [Project Euler #47](https://www.hackerrank.com/contests/projecteuler/challenges/euler047/problem?isFullScreen=true)

## Problem

The first two consecutive numbers to have two distinct prime factors are:

$14 = 2*7$
$15 = 3*5$

The first three consecutive numbers to have three distinct prime factors are:

$644 = 2^2*7*23$
$645 = 3*5*43$
$646 = 2*17*19$

Given N, find all consecutive integers, where first integer is $<=N$ to have exactly K distinct prime factors. Print the first of these numbers in ascending order.

### Input format

```N K```

### Output format

Print the answer corresponding to the test case. Print each integer in a new line.

### Constrains

$20<=N<=2*10^6$
$2<=K<=4$

## Solutions

### Basic way

The first logic may comes into our mind is we will iterate though all elements from 1 to N, check if current is element have exactly K distinct prime factors. If that is true, we will check `K - 1` next element to check that conditions. If the next `K - 1` elements satisfy the condition, print the first element.

Of course this solution is too slow, even if we optimize some part like we will save the prime numbers in a list (numbers have zero distinct prime factors), and then, base on that list, we will count the number of distinct prime factors.

```py
import math

data = input().split(' ')

list_prime = [2, 3, 5, 7]
n = int(data[0])
k = int(data[1])
first_number = 0
count_consecutive = 0

# Function count distinct prime factors
# If a number has 0 prime factors, that number is prime number and will be pushed to list_prime
# Algorithm: Count how many prime number in list_prime such that the input number is divisible by.
# We will iterate though the list prime, if the current prime is greater than num / 2 then we will stop
def count_divisors(num):
    divisors = {}
    _max = math.floor(num / 2)

    for prime in list_prime:
        # if prime is greater than num / 2 => don't need to check anymore
        if prime > _max:
            break

        if num % prime == 0:
            divisors[str(prime)] = 1

    # if number of distinct prime factors is 0, that number is prime number and we will push that number into list prime
    if len(divisors.keys()) == 0:
        list_prime.append(num)

    return len(divisors.keys())


for i in range(12, n + 1, 1):
    if count_divisors(i) != k:
        first_number = 0
        count_consecutive = 0
        continue

    if first_number == 0:
        first_number = i

    count_consecutive += 1

    if count_consecutive == k:
        print(first_number)
        first_number = 0
        count_consecutive = 0

# In case existing the number have k distinct prime numbers, but it reach the max n
# we need to check the next k - count_consecutive numbers
if first_number != 0:
    for i in range(n + 1, n + (k - count_consecutive) + 1, 1):
        if count_divisors(i) == k:
            count_consecutive += 1

    if count_consecutive == k:
        print(first_number)
```

### Optimal solution

You can see that the basic way is way too slow. To optimize the algorithm, we will use **Sieve of Eratosthenes** algorithm. You can read more about this in [this link](https://depressed-developer.vercel.app/detail/algorithm/4)

The idea is we will create an array with `n + k` elements. And we will count the number of distinct primes for each elements (using sieve of Eratosthenes algorithm). And then, we will iterate though the array, and check if exist a sub array such that the number of distinct primes for each element in that array is equal to `k`.

Example: `n = 20`, `k = 2`

![Count number of prime factors](https://raw.githubusercontent.com/tienmanh2208/public-asset-depressed-developer/master/posts/algorithm/distinct_prime_factors/count_prime_factors.png)

The above pictures illustrates the result we have after counting prime factors for each number (number is the index). You can see that there are two sub arrays that satisfy the condition (each element has two prime factors) is `(14, 15)` and `(20, 21)`, so we will print the first element of each array is `14` and `20`

```py
data = input().split(' ')
n = int(data[0])
k = int(data[1])

arr_numbers = [0 for i in range(n + 4)]
arr_numbers[0] = -10
arr_numbers[1] = -10

# Use Sieve of Eratosthenes Method to find prime numbers
for i in range(2, n + 4, 1):
    if arr_numbers[i] != 0:
        continue

    for j in range(i * 2, n + 4, i):
        arr_numbers[j] += 1

list_prime = []

for i in range(10, n + 1, 1):
    if arr_numbers[i] != k:
        continue

    flag = True

    for j in range(i + 1, i + k, 1):
        if arr_numbers[j] != k:
            flag = False

    if flag:
        print(i)
```
