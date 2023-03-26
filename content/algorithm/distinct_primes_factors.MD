https://www.hackerrank.com/contests/projecteuler/challenges/euler047/problem?isFullScreen=true

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