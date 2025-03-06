---
title: "Dynamics of a dynamic array"
date: 2025-03-06
description: "memory, fibonacci, phi"
tags: ["life"]
---

{{< lead >}}
Memory and this beauty: \\(\varphi = \dfrac{1+\sqrt5}{2}= 1.6180339887…\\)
{{< /lead >}}

{{< katex >}}

## Changing sizes
If you do programming you would be familiar with the likes of `std::vector` or `ArrayList<>` or `slices`. These are examples of dynamic array.
These are arrays which can change their size during runtime. On the contrary we have static arrays which have their size defined which cannot be changed or resized.
A natural question is how do these structures grow in size when elements are appended to them?

Consider the following terms:
- size: logical length of the array
- capacity: the maximum logical length of the array
```c
int arr[256]; // an static array of capacity 256 is defined
arr[0] = 1;   // size of the array is 2 
arr[1] = 2;   // as only 2 meaningful elements are inserted
```

The approach used goes like this
- use a static array
- *grow* the array in size when inserting a new element makes size >= capacity
- this is is done by creating a new array of larger capacity and copying the older array in this new memory location
- the older array is then deleted and this new larer array can be used for further appends
- but the question is, how large this new array should be?

The below code snippet shows the approach used with **doubling** the array when required-
```c
function insertEnd(dynarray a, element e)
    if (a.size == a.capacity)
        // notice the capacity is doubled here
        a.capacity = a.capacity * 2 
        // (copy the contents to the new memory location here)

    a[a.size] = e       // assign e to last index
    a.size = a.size + 1 // increase size
```

## Growth Factor
Now to answer, what should be optimal growth factor for this operation? We first need to define the cost of the operation for this question to make sense.
To understand this, let's take a look at what factor existing systems use[^1]

| Implementation          | Growth Factor |
| ---                     | ---           |
| Java `ArrayList`        | 1.5 | 
| Python `PyListObject`   | ~1.125 (n + (n >> 3)) | 
| G++ 5.2.0               | 2 | 
| Facebook folly/FBVector | 1.5 | 
| Go slices               | between 1.25 and 2 |

The folly/FBVector[^5] describes why growth factor of 2 is not good:

Say the size of array is \\(S\\), capacity \\(C\\) and growth factor \\(f\\)
- first reallocation, a new array of capacity \\(f\*C\\) is created while the old memory of size \\(C\\) is marked obsolete
- second reallocation, a new array of capacity \\(f\*f\*C\\) is created while older memory of size \\(C + f\*C\\) is marked obsolete
- since arrays are contiguous we are constrained to only take contiguous sections in the memory

This creates the following series of memory-
$$
 C, C\*f, C\*f^2, C\*f^3, ...
$$

Say \\(f = 2\\), this presents the following series-
$$
 C, 2C, 4C, 8C, ...
$$

Here every element in the series will be strictly larger than the sum of all previous ones because of the remarkable equality:
$$
 C + 2C + 4C + 8C + ... + 2^nC = (2^{n+1} - 1)C < 2^{n+1}C
$$

So this means new array is of size \\(2^{n+1}C\\) and cannot use the obsolete marked memory because it is of size \\((2^{n+1}-1)C\\), hence a growth factor less than 2 is preferred.

This can also be demonstrated by taking some real values, say growth factor is 1.5, and we start with 16 byte array allocation
- Start with 16 bytes
- When you need more, allocate 24 bytes, then free up the 16, leaving a 16-byte hole
- When you need more, allocate 36 bytes, then free up the 24, leaving a 40-byte hole
- When you need more, allocate 54 bytes, then free up the 36, leaving a 76-byte hole
- When you need more, allocate 81 bytes, then free up the 54, leaving a 130-byte hole
- When you need more, use 122 bytes (rounding up) from the 130-byte hole
- This will not be possible with a growth factor of 2, your array will be forced to crawl forward in memory and will not be able to use free-ed up memory.

## Optimal Growth Factor
For optimality, we need to use up the space free-ed up, as soon as possible.
This will happen after the second reallocation[^2] if and only if obsolete memory size is big enough to contian new memory
$$
C + f\*C >= f\*f\*C
$$
The \\(C\\) factor can be cancelled out here
$$
1 + f <= f^2
$$
$$
f^2 - f - 1 >= 0
$$
$$
f <= \varphi = \frac{1 + \sqrt{5}} {2}
$$

And yes the golden ratio just made its way in this problem as well. This reminds of a very beautiful visual[^4] shown in Numberphile regarding using golden ratio.

I wrote this whole page because [this](https://www.youtube.com/watch?v=GZPqDvG615k) video popped up in my youtube feed. Very interesting insights which I have not mentioned in this post.


[^1]: [Dynamic Array - Wikipedia](https://en.wikipedia.org/wiki/Dynamic_array)
[^2]: [cpp-internals-stl-vector](https://web.archive.org/web/20150806162750/http://www.gahcep.com/cpp-internals-stl-vector-part-1/)
[^3]: [Why Phi is most irrational - Math Stackexhange](https://math.stackexchange.com/questions/395938/why-is-varphi-called-the-most-irrational-number)
[^4]: [The Golden Ratio (why it is so irrational) - Numberphile](https://www.youtube.com/watch?v=sj8Sg8qnjOg)
[^5]: [Facebook folly/FBVector](https://github.com/facebook/folly/blob/main/folly/docs/FBVector.md)

