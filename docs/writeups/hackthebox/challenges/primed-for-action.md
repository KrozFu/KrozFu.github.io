# Primed for Action Challenge

### Context

Intelligence units have intercepted a list of numbers. They seem to be used in a peculiar way: the adversary seems to be sending a list of numbers, most of which are garbage, but two of which are prime. These 2 prime numbers appear to form a key, which is obtained by multiplying the two. Your answer is the product of the two prime numbers. Find the key and help us solve the case.

### Solution

#### Requirements

- python

#### Step 1 - Analyze the code

The code gives us a list of numbers, which are to search for prime numbers and when they are found, multiply the two numbers.

```bash
4291 4038 403 2832 4323 3368 2607 2571 1527 1205 1929 1092 2883 4658 1379 4152 2148 294 1573 2584 2990 3244 254 2210 544 670 4047 3942 1446 217 3822 3114 3075 4877 3321 4459 4514 4364 1881 2958 1456 375 2911 1917 3861 3753 4311 3577 3149 3467
```

#### Step 2 - Create code python

In the following Python code, a search is performed for prime numbers in the list of prime numbers, and then when they are found, they are taken out in a sublist, and then they are multiplied.

```python
# calculate answer
def cal_prime(n):
    if n < 2:
        return False
    for i in range (2, int(n**0.5) + 1):
        if n % i == 0:
            return False
    return True

# take in the number
n = input()

# List numbers
numbers = list(map(int, n.split()))

# Find prime list
prime_numbers = [num for num in numbers if cal_prime(num)]

# Cal product prime
if len(prime_numbers) == 2:
    product = prime_numbers[0] * prime_numbers[1]
    print(product)
else:
    print("Does not comply.")
```

The output shows us the list of numbers.

```bash
4291 4038 403 2832 4323 3368 2607 2571 1527 1205 1929 1092 2883 4658 1379 4152 2148 294 1573 2584 2990 3244 254 2210 544 670 4047 3942 1446 217 3822 3114 3075 4877 3321 4459 4514 4364 1881 2958 1456 375 2911 1917 3861 3753 4311 3577 3149 3467
```

In the end, the result of the multiplication that leads us to the challenge flag.

```bash
16908559
```

#### Step 3 - Flag

```bash
HTB{pr1m3_Pr0}
```

#### Step 4 - Conclusion

In this challenge, the programming part was practiced and the ability to identify the reasoning logic of prime numbers was achieved.
