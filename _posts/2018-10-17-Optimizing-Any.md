---
layout: post
title: Optimizing the any function in python
category: Coding
---

Recently when writing code I came across the fact that the `any` function in python is not optimized on list comprehensions. This can be demonstrated with the following code:

```python
import time
start = time.time()
any([True for x in range(10 ** 8)])
end = time.time()
print(round(end - start, 2))
```

And here is the output:

```
12.28
```

This is very slow! I would hope this would be optimized to convert the list comprehension so that after the first `True` is generated the whole function returns `True`. When I saw this, I decided to try my own implementation of the `any` function that short circuits a list comprehension, and managed to do this using lambdas.

```python
def fast_any(my_l):
	'''
	my_l must be a list of () -> Boolean
	'''
	for func in my_l:
		if func():
			return True

	return False
```

To be clear, the entire list comprehension will still execute. However, the values in the list comprehension are evaluated lazily. This is great if the function to generate the values in the list comprehension will take a long time to run. To show this was faster than the stock `any`, I wrote the following:

```python
import time
def fast_any(my_l):
	'''
	my_l must be a list of () -> Boolean
	'''
	for func in my_l:
		if func():
			return True

	return False

def function_takes_time():
	time.sleep(0.01)
	return True

start = time.time()
print(any([function_takes_time() for i in range(10 ** 3)]))
end = time.time()
print("Time to run any on list comprehension size 10 ** 3: {}".format(round(end - start, 2)))

start = time.time()
print(fast_any([lambda: function_takes_time() for i in range(10 ** 3)]))
end = time.time()
print("Time to run fast_any on list comprehension size 10 ** 3: {}".format(round(end - start, 2)))
```

And here was my output:

```
True
Time to run any on list comprehension size 10 ** 3: 10.73
True
Time to run fast_any on list comprehension size 10 ** 3: 0.01
```

This is definitely a big improvement. But what if I want my function to actually use the `i` that is given? I ran into the problem that python, like most languages, is lexically scoped. This is demonstrated using the following:

```python
[f() for f in [lambda: x for x in range(4)]]
```

Output:

```
[4, 4, 4, 4, 4]
```

In order to fix this, we have to use even more lambdas. The following will output what we want:

```python
[(lambda z: (lambda: z))(x)() for x in range(5)]
```

Output:

```
[0, 1, 2, 3, 4]
```

The reason this gives us what we want is because we are calling the function on the `x` variable in the list comprehension. See the following stackoverflow post for a much better explanation: [https://stackoverflow.com/a/34021333](https://stackoverflow.com/a/34021333). Now let's show this working with our new `fast_any` function!

```python
import time
def fast_any(my_l):
	'''
	my_l must be a list of () -> Boolean
	'''
	for func in my_l:
		if func():
			return True

	return False

def function_takes_time(i):
	time.sleep(0.01)
	return i > 500

start = time.time()
print(any([function_takes_time(i) for i in range(10 ** 3)]))
end = time.time()
print("Time to run any on list comprehension size 10 ** 3: {}".format(round(end - start, 2)))

start = time.time()
print(fast_any([(lambda x: (lambda: function_takes_time(x)))(i) for i in range(10 ** 3)]))
end = time.time()
print("Time to run fast_any on list comprehension size 10 ** 3: {}".format(round(end - start, 2)))
```

Output:

```
True
Time to run any on list comprehension size 10 ** 3: 10.89
True
Time to run fast_any on list comprehension size 10 ** 3: 5.47
```

Our fast_any properly only goes through half of the list! Once it finds a function that returns true, the function returns true. This functionality can also be extended to the `all` function, and probably many others, but this demonstrates a powerful way to speed up your python code, while keeping it somewhat "pythonic".