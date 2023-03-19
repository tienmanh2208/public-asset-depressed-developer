https://www.hackerrank.com/contests/projecteuler/challenges/euler015/problem?isFullScreen=true

```python
# Enter your code here. Read input from STDIN. Print output to STDOUT
t = int(input())
divisor = 10 ** 9 + 7

stat = []
for i in range(501):
    stat.append([0 for x in range(501)])

for i in range(501):
    stat[0][i] = 1
    stat[i][0] = 1
    
for i in range(1, 501, 1):
    for j in range(1, 501, 1):
        stat[i][j] = stat[i - 1][j] + stat[i][j - 1]
        
for i in range(t):
    size = input().split(' ')
    rows = int(size[0])
    columns = int(size[1])
    print(stat[rows][columns] % divisor)
```