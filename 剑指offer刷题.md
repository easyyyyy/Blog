# 剑指offer

## 整数

### 整数除法

- 减法代替除法 O(n)
- 优化时间复杂度 每次把除数翻倍

```js
var divide1 = function(a, b) {
    const INT_MIN = -Math.pow(2, 31)
    const INT_MAX = Math.pow(2, 31) - 1

    if (a == INT_MIN && b == -1) return INT_MAX

    const sign = (a > 0) ^ (b > 0) ? -1 : 1
    if (a > 0) a = -a 
    if (b > 0) b = -b 

    let res = 0
    while (a <= b) {
        let value = b
        let k = 1
        //被除数是否大于除数的2倍 是的话判断是否大于4倍 以此类推
        while (value >= 0xc0000000 && a <= value + value) {
            value += value
            k += k
        }
        a -= value
        res += k
    }
    return sign == 1 ? res : -res
};
```

- 位运算

比如，a=22，b=3

可以每次从最大位数开始尝试

22 - (3 << 31) < 0

22 - (3 << 30) < 0

.....

22 - (3<<2) = 10 > 0

10 - (3<<1) = 4 > 0

4 - (3 << 0) = 1 > 0

22/3 = (1<<2) + (1<<1) + （1<<0）= 4 + 2 + 1 = 7

时间复制度为O(31) = O(1)

```js

var divide = function(a, b) {
    const INT_MIN = -Math.pow(2, 31)
    const INT_MAX = Math.pow(2, 31) - 1

    if (a == INT_MIN && b == -1) return INT_MAX

    const sign = (a > 0) ^ (b > 0) ? -1 : 1
    a = Math.abs(a)
    b = Math.abs(b)

    let res = 0
    
    for (let x = 31; x >= 0; x--) {
        // 首先，右移的话，再怎么着也不会越界
        // 其次，无符号右移的目的是：将 -2147483648 看成 2147483648

        // 注意，这里不能是 (a >>> i) >= b 而应该是 (a >>> i)... - b >= 0
        // 这个也是为了避免 b = -2147483648，如果 b = -2147483648
        // 那么 (a >>> i) >= b 永远为 true，但是 (a >>> i) - b >= 0 为 false
        if ((a >>> x) - b >= 0) {
            a = a - (b << x)
            res = res + (1 << x)
        }
    }
    if (res == -2147483648) return -2147483648
    return sign == 1 ? res : -res
};
```



