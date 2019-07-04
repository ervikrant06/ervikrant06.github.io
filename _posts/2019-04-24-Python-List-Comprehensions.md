---
layout: post
title: Python list/dict comprehensions tips and trics
tags: [Python]
category: [Python]
author: vikrant
comments: true
--- 


In this article, I am sharing some information related to python comprehension. I always wanted to use them while writing python code but always gets messed up with SYNTAX hence this is also kinda cheat sheet for me to use python comprehension next time. 

If you are fan of `map` I would suggest to read some discussions on stackoverflow about the comparison of map with comprehension.

#### List

- List with if condition. Example: Get the even numbers from the list. Traverse through the list and use if condition to see the even numbers. 

```
$ python
listPython 3.5.2 (default, Nov 12 2018, 13:43:14) 
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> list1=[1,2,3,4,5,6,7,8]
>>> even = [i for i in list1 if i%2 == 0]
>>> even
[2, 4, 6, 8]
```

- List with if/else condition. Example: Get both odd and even numbers and this time replace the oddnumber with `odd` similarly evennumber with `even`.

```
>>> odd_even = [ 'even' if i%2 == 0 else 'odd' for i in list1 ]
>>> odd_even
['odd', 'even', 'odd', 'even', 'odd', 'even', 'odd', 'even']
```

- List with if/elif/else condition. Example: Mark number less than 2 as `less_than_two` and number greater than two as `greater_than_two` and zero as `zero`. This SYNTAX is bit complex but take the reference of if/else condition to understand it. 

```
>>> list1
[1, 2, 3, 4, 5, 6, 7, 8]
>>> list1.append(0)
>>> list1
[1, 2, 3, 4, 5, 6, 7, 8, 0]
>>> first_two = [ 'greater_than_two' if i > 2 else 'zero' if i == 0 else 'less_than_two' for i in list1 ]
>>> print(first_two)
['less_than_two', 'less_than_two', 'greater_than_two', 'greater_than_two', 'greater_than_two', 'greater_than_two', 'greater_than_two', 'greater_than_two', 'zero']
```

- Nested List with if. Example: Traverse  through the list and if the element is list then run traverse through this list otherwise don't take any action. out list contains all nested list elements except 7 which is not a list. 

```
>>> nested_list = [[1,2],[3,4],[5,6],7]
>>> [nest_i for i in nested_list if isinstance(i, list) for nest_i in i]
[1, 2, 3, 4, 5, 6]
```

- Nested list with if/else. Example: Continuing from the previous example. 

```
>>> ['even' if nest_i%2 == 0 else 'odd' for i in nested_list  if isinstance(i, list) for nest_i in i]
['odd', 'even', 'odd', 'even', 'odd', 'even']
```

#### Dictionary

- Dictionary with if condition. Example: Traverse through the dict and if the key is `key2` then only print the key,value pair. 

```
>>> dict1={'key1':'value1','key2':'value2'}
>>> {k:v for k,v in dict1.items() if k == 'key2'}
{'key2': 'value2'}
```

- Dictionary with if/else condition. Example: Traverse through the dict and if the key is `key2` then use the existing k:v pair otherwise add new one. 

```
>>> {k:v if k == 'key2' else {'key3':'value3'} for k,v in dict1.items() }
{'key1': {'key3': 'value3'}, 'key2': 'value2'}
>>> 

>>> {k:('random' if v == 'value2' else v) for k,v in dict1.items() }
{'key1': 'value1', 'key2': 'random'}
>>> 
```

- Dictionary with if/else/if condition. Example: if key is key1 or key4  then replace the value of that key with new k:v pair.

```
>>> dict1={'key1':'value1','key2':'value2','key5':'value5'}
>>> {k:v if k == 'key2' else {'key3':'value3'} if k == 'key1' else {'key4':'value4'} for k,v in dict1.items() }
{'key1': {'key3': 'value3'}, 'key2': 'value2', 'key5': {'key4': 'value4'}}
```

- Nested dictionary with if condition. Example: if value is a dict then print the k:v pair. 

```
nested_dict = { 'dictA': {'key_1': 'value_1'},
...                 'dictB': {'key_2': 'value_2'}}

>>> {nest_k: nest_v for k,v in nested_dict.items() if isinstance(v, dict) for nest_k,nest_v in v.items()}
{'key_1': 'value_1', 'key_2': 'value_2'}
```

- Nested dict with if/else condition. Example if the value of nested  dict is value_2 then replace that value with some other value.

```
>>> {nest_k:('random' if nest_v == 'value_2' else nest_v) for k,v in nested_dict.items() if isinstance(v,dict) for nest_k,nest_v in v.items()}
{'key_1': 'value_1', 'key_2': 'random'}
>>> 
```