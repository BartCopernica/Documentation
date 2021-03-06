# Lambda functions

C++ and PHP both support lambda functions or anonymous functions (in 
the C++ world the word 'lambda' is most used, PHP people speak about
'anonymous functions'). With PHP-CPP you can pass these functions 
from one language to the other. It is possible to call an anonymous 
PHP function from your C++ code, and the other way around, to call a C++
lambda from a PHP script.

## Calling anonymous PHP functions from C++

Let's start with a very simple example in PHP. In PHP you can create
anonymous functions, and assign them to a variable (or pass them
directly to a function).

```php
<?php
// anonymous PHP function stored in the variable $f
$f = function($a, $b) {
    
    // return the sum of the parameters
    return $a + $b;
};

// pass the function to another function
other_function($f);

// or pass an anonymous function without assigning it to a variable
other_function(function() {
    
    // return the product of the parameters
    return $a * $b;
});

?>
```

The code above should be familiar to most PHP programmers. The 
'other_function' can of course be implemented in PHP user space,
but to demonstrate how to do this with PHP-CPP we are going to
build it with C++. Just like all the other functions that you've
seen in the earlier examples, such a C++ function function receives 
a Php::Parmeters object as its parameter, which is a std::vector of 
Php::Value objects.

```cpp
#include <phpcpp.h>
/**
 *  Native function that is callable from PHP
 *
 *  This function gets one parameter that holds a callable anonynous
 *  PHP function.
 *
 *  @param  params      The parameters passed to the function
 */
void other_function(Php::Parameters &params)
{
    // make sure the function was really called with at least one parameter
    if (params.size() == 0) return nullptr;

    // this function is called from PHP user space, and it is called
    // with a anonymous function as its first parameter
    Php::Value func = params[0];
    
    // the Php::Value class has implemented the operator (), which allows
    // us to use the object just as if it is a real function
    Php::Value result = func(3, 4);
    
    // @todo do something with the result
}

/**
 *  Switch to C context, because the Zend engine expects the get_module()
 *  to have a C style function signature
 */
extern "C" {
    /**
     *  Startup function that is automatically called by the Zend engine
     *  when PHP starts, and that should return the extension details
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() 
    {
        // the extension object
        static Php::Extension extension("my_extension", "1.0");
        
        // add the example function so that it can be called from PHP scripts
        extension.add("other_function", other_function);
        
        // return the extension details
        return extension;
    }
}
```
It is that simple. But the other way around is possible too. Imagine
we have a function in PHP user space code that accepts a callback 
function. The following function is a simple version of the 
PHP array_map() function:

```php
<?php
// function that iterates over an array, and calls a function on every
// element in that array, it returns a new array with every item
// replaced by the result of the callback
function my_array_map($array, $callback) {
    
    // initial result variable
    $result = array();
    
    // loop through the array
    foreach ($array as $index => $item) {
        
        // call the callback on the item
        $result[$index] = $callback($item);
    }
    
    // done
    return $result;
}
?>
```

Imagine that we want to call this PHP function from your C++ code,
using a C++ lambda function as a callback. This is possible, and easy:

```cpp
#include <phpcpp.h>
/**
 *  Native function that is callable from PHP
 */
void run_test()
{
    // create the anonymous function
    Php::Function multiply_by_two([](Php::Parameters &params) -> Php::Value {
        
        // make sure the function was really called with at least one parameter
        if (params.size() == 0) return nullptr;
        
        // one parameter is passed to the function
        Php::Value param = params[0];
        
        // multiple the parameter by two
        return param * 2;
    });

    // the function now is callable
    Php::Value four = multiply_by_two(2);
    
    // a Php::Function object is a derived Php::Value, and its value can 
    // also be stored in a normal Php::Value object, it will then still 
    // be a callback function then
    Php::Value value = multiply_by_two;
    
    // the value object now also holds the function
    Php::Value six = value(3);
    
    // create an array
    Php::Value array;
    array[0] = 1;
    array[1] = 2;
    array[2] = 3;
    array[3] = 4;
    
    // call the user-space function
    Php::Value result = Php::call("my_array_map", array, multiply_by_two);
    
    // @todo do something with the result variable (which now holds
    // an array with values 2, 4, 6 and 8).
}

/**
 *  Switch to C context, because the Zend engine expects the get_module()
 *  to have a C style function signature
 */
extern "C" {
    /**
     *  Startup function that is automatically called by the Zend engine
     *  when PHP starts, and that should return the extension details
     *  @return void*
     */
    PHPCPP_EXPORT void *get_module() 
    {
        // the extension object
        static Php::Extension extension("my_extension", "1.0");
        
        // add the example function so that it can be called from PHP scripts
        extension.add("run_test", run_test);
        
        // return the extension details
        return extension;
    }
}
```
In the example we assigned a C++ lambda function to a Php::Function
object. The Php::Function class is derived from the Php::Value class.
The only difference between a Php::Value and a Php::Function is
that the constructor of Php::Function accepts a function. Despite 
that difference, both classes are completely identical. In fact, we 
would have preferred to make it possible to assign C++ functions 
directly to Php::Value objects, and skip the Php::Function 
constructor, but that is impossible because of calling ambiguities.

The Php::Function class can be used as if it is a normal Php::Value
object: you can assign it to other Php::Value objects, and you
can use it as a parameter when you call user space PHP functions.
In the above example we do exactly that: we call the user space
my_iterate() function with our own 'multiply_by_two' C++ function.

## Signature of the C++ function

You can pass different sort of C++ functions to the Php::Function
constructor, as long as they are compatible with the following two
function signatures:

```cpp
Php::Value function();
Php::Value function(Php::Parameters &params);
```

Internally, the Php::Function class uses a C++ std::function object 
to store the function, so anything that can be stored in such a 
std::function object, can be assigned to the Php::Function class.
