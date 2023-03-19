https://www.hackerrank.com/contests/projecteuler/challenges/euler004/problem?isFullScreen=true

```python
t = int(input())

def isPalindromicNumber(num):
    strNum = str(num)
    if len(strNum) != 6:
        return False
    
    return strNum[0] == strNum[5] and strNum[1] == strNum[4] and strNum[2] == strNum[3]

statValue = {}

# Calculate all the product of two numbers from 100 to 999
# Filter palindromic numbers from these results.
for i in range(100, 999, 1):
    for j in range(100, 999, 1):
        num = i * j;
        if isPalindromicNumber(num):
            statValue[str(num)] = 1

stat = [int(x) for x in list(statValue.keys())]
stat.sort()

for i in range(t):
    n = int(input())
    
    for index, num in enumerate(stat):
        if n <= num:
            print(stat[index - 1])
            break;
```