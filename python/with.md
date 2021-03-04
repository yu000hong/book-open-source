# with statement in Python

> `with` statement in Python is used in exception handling to make the code cleaner and much more readable. It simplifies the management of common resources like file streams. The `with` statement itself ensures proper acquisition and release of resources. Thus, `with` statement helps avoiding bugs and leaks by ensuring that a resource is properly released when the code using the resource is completely executed. 

```python
# without using with statement 
file = open('file_path', 'w') 
try: 
    file.write('hello world') 
finally: 
    file.close() 
```

```python
# using with statement 
with open('file_path', 'w') as file: 
    file.write('hello world !')
```

### Supporting the “with” statement in user defined objects

To use with statement in user defined objects you only need to add the methods `__enter__()` and `__exit__()` in the object methods. 

```python
class MessageWriter(object): 
    def __init__(self, file_name): 
        self.file_name = file_name 
      
    def __enter__(self): 
        self.file = open(self.file_name, 'w') 
        return self.file
  
    def __exit__(self): 
        self.file.close() 
  
# using with statement with MessageWriter 
with MessageWriter('my_file.txt') as xfile: 
    xfile.write('hello world') 
```

As soon as the execution enters the context of the with statement a MessageWriter object is created and python then calls the `__enter__()` method. In this `__enter__()` method, initialize the resource you wish to use in the object. This `__enter__()` method should always return a descriptor of the acquired resource.

As soon as the code inside the with block is executed, the `__exit__()` method is called. All the acquired resources are released in the `__exit__()` method. This is how we use the with statement with user defined objects.

This interface of `__enter__()` and `__exit__()` methods which provides the support of with statement in user defined objects is called **Context Manager**.

### The `contextlib` module

A class based context manager as shown above is not the only way to support the with statement in user defined objects. The `contextlib` module provides a few more abstractions built upon the basic context manager interface. Here is how we can rewrite the context manager for the MessageWriter object using the contextlib module.

```python
from contextlib import contextmanager 
  
class MessageWriter(object): 
    def __init__(self, filename): 
        self.file_name = filename 
  
    @contextmanager
    def open_file(self): 
        try: 
            file = open(self.file_name, 'w') 
            yield file
        finally: 
            file.close() 
  
# usage 
message_writer = MessageWriter('hello.txt') 
with message_writer.open_file() as my_file: 
    my_file.write('hello world') 
```

In this code example, because of the `yield` statement in its definition, the function `open_file()` is a **generator function**.

When this open_file() function is called, it creates a resource descriptor named file. This resource descriptor is then passed to the caller and is represented here by the variable my_file. After the code inside the with block is executed the program control returns back to the open_file() function. The open_file() function resumes its execution and executes the code following the yield statement. This part of code which appears after the yield statement releases the acquired resources. The `@contextmanager` here is a decorator.

The previous **class-based** implementation and this **generator-based** implementation of context managers is internally the same. While the later seems more readable, it requires the knowledge of generators, decorators and yield.
