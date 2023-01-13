# $N^{th}$ prime number

*Source: [10001st prime](https://www.hackerrank.com/contests/projecteuler/challenges/euler007/problem?isFullScreen=true)*

## Problem

What is $N^{th}$ prime number?

**Input Format**

First line contains T that denotes the number of test cases. This is followed by T lines, each containing an integer, N.

**Constraints**

- $1 \le T \le 10^3$
- $1 \le N \le 10^4$

**Output Format**

Print the required answer for each test case.

## Solution

```js
// arr contains T test cases
function main(arr) {
    let listPrimeNum = [BigInt(2), BigInt(3)];
    arr.forEach((n) => {
        if (listPrimeNum.length >= n) {
            console.log(listPrimeNum[n - 1].toString())
            continue;
        }
        
        let startPoint = BigInt(1);
        if (listPrimeNum.length > 0) {
            startPoint = listPrimeNum[listPrimeNum.length - 1] + BigInt(2)
        }

        while(listPrimeNum.length < n) {
            while(true) {
                if (isPrime(startPoint, listPrimeNum)) {
                    listPrimeNum.push(startPoint);
                    break;
                }
                
                startPoint += BigInt(2);
            }
        }
        
        console.log(listPrimeNum[listPrimeNum.length - 1].toString())
    })
}

function isPrime(number, listPreviousPrime) {
    if (number == 1) {
        return false;
    }
    
    for(let i = 0; i < listPreviousPrime.length; ++i) {
        if (number % listPreviousPrime[i] == 0) {
            return false;
        }
    }
    
    return true;
}
```
