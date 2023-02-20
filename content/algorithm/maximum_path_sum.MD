https://www.hackerrank.com/contests/projecteuler/challenges/euler067/problem?isFullScreen=true

Recursion

```js
function processData(input) {
    input = input.split('\n');
    const t = parseInt(input.shift());

    for (let i = 0; i < t; ++i) {
        const height = parseInt(input.shift());

        const triangle = input.splice(0, height).map((row) => row.split(' ').map((each) => parseInt(each)))

        if (triangle.length === 1) {
            console.log(triangle[0][0])
        } else {
            console.log(triangle[0][0] + maxTotal(1, 0, triangle));
        }
    }
}

function maxTotal(currentRow, preIndex, triangle) {
    let left, right;
    if (currentRow === triangle.length - 1) {
        left = triangle[currentRow][preIndex];
        right = triangle[currentRow][preIndex + 1];
    } else {
        left = triangle[currentRow][preIndex] + maxTotal(currentRow + 1, preIndex, triangle);
        right = triangle[currentRow][preIndex + 1] + maxTotal(currentRow + 1, preIndex + 1, triangle);
    }

    return left > right ? left : right;
}
```

de-recursion (More optimizing)

```js
function processData(input) {
    input = input.split('\n');
    const t = parseInt(input.shift());

    for (let i = 0; i < t; ++i) {
        const height = parseInt(input.shift());

        const triangle = input.splice(0, height).map((row) => row.split(' ').map((each) => parseInt(each)))

        if (triangle.length > 1) {
            for (let j = triangle.length - 2; j >= 0; --j) {
                for (let k = 0; k < triangle[j].length; ++k) {
                    triangle[j][k] += Math.max(triangle[j + 1][k], triangle[j + 1][k + 1]);
                }
            }
        }
        
        console.log(triangle[0][0])
    }
}
```
