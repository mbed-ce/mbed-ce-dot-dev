# Callback

In software development, it is useful to be able to pass a reference to your own function to another function or class, so that your code can be executed later. This is called a callback, and is a common concept in most programming languages.

Unfortunately, C++ makes callbacks a bit more difficult than necessary, especially when you want to pass in a member function of a class (member function pointers have a fundamentally different type than regular global function pointers). In desktop C++, there exists the `std::function` class to wrap this complexity, but this class isn't a good fit for embedded development -- it pulls in a large amount of code from the standard library and uses dynamic allocation.

This is where the [`mbed::Callback`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_callback_3_01_r_07_arg_ts_8_8_8_08_4.html) class comes in -- it provides a much more lightweight way to wrap C/C++ function pointers that is compatible with embedded development.

## Overview of Callback

The `mbed::Callback` class can wrap a global function, member function, or lambda function, and treats all three kinds equally. So, for instance, if you define a `Callback<int(float)>`, then you can create that callback from any function that takes a float and returns an int, regardless of whether it's a global, member, or lambda function. Type safety is still guaranteed via the template argument system, so you shouldn't have to worry about passing a function of the wrong type.

Callbacks cannot store any parameter values internally when the callback is created (this is often referred to as a "partial function" (python) or "parameter binding" (desktop C++) or "currying" (generic term)). However, parameters can be passed through when the function is created without constraints, and the return value is passed back as well.

```cpp
Callback<int(bool, char*)> cbWithParams = ...;
int result = cbWithParams(true, "some string");
```

## Creating Callbacks

This section will show you how to actually instantiate a callback based on the type of function you are working with.

Up until C++17 at least, template arguments could not be deduced for object constructors. So, Mbed provides the `mbed::callback` (lower case c) global function to create a callback while deducing the type of the arguments. This function can often provide a more concise way to create callbacks (at the cost of somewhat more complex compiler errors if something goes wrong) if you don't have a typedef for the function name. So, it will be illustrated here as well.

For this section, suppose we have the following code:
```cpp
typedef Callback<bool(uint8_t, char const *)> EventHandler; // Handles an event. Args are event code and description message, return value is true for success

void addEventHandler(EventHandler const & event) {
    ...
}
```

Here's how to create callbacks using this API:

### For a Global Function

```cpp
bool globalHandler(uint8_t code, char const * message) {
    ...
}
```

For this, we just need to pass the function name:

```cpp
addEventHandler(mbed::callback(globalHandler));
addEventHandler(EventHandler(globalHandler));
addEventHandler(Callback(globalHandler)); // C++17 or later only
```

!!! note
    You can also explicitly take the address of `globalHandler` by putting an &amp; in front of it, it works either way.

### For a Member Function

```cpp
class SomeClass {
public:
    bool memberHandler(uint8_t code, char const * message) {
        ...
    }
};

SomeClass instance;
```

For member functions it's a little more annoying. You have to pass the instance pointer, and then the class and function name.

```cpp
addEventHandler(mbed::callback(&instance, &SomeClass::memberHandler));
addEventHandler(EventHandler(&instance, &SomeClass::memberHandler));
addEventHandler(Callback(&instance, &SomeClass::memberHandler)); // C++17 or later only
```

!!! warning
    When passing member functions, the &amp; is required in front of the second argument (for some reason...)

Note that if you are adding the callback from within your class, you would use the this-pointer:

```cpp
addEventHandler(mbed::callback(this, &SomeClass::memberHandler));
```

### For a Lambda

Now suppose we have a lambda we want to register a callback for:

```cpp
auto lambdaHandler = [&](uint8_t code, char const * message) {
    ...
    return false;
};
```

We can do that by just passing in the lambda variable to the callback constructor:

```cpp
addEventHandler(mbed::callback(lambdaHandler));
addEventHandler(EventHandler(lambdaHandler));
addEventHandler(Callback(lambdaHandler)); // C++17 or later only
```

**However**, there is an important consideration to be aware of when using capturing lambdas. Each data member captured into the lambda causes its size to increase, and there is only a fixed, pre-determined amount of storage available for lambda data members inside the Callback. This is currently three machine words, i.e. enough to capture three variables by reference or capture three values of machine word size by value.

So, the following code:

```cpp
volatile bool flag1, flag2, flag3, flag4;
...
auto tooBigLambda = [&](uint8_t code, char const * message) {
    return flag1 && flag2 && flag3 && flag4;
};
EventHandler handler(tooBigLambda);
```

yields a compiler error like

```
mbed-os/platform/include/platform/Callback.h:640:33: error: static assertion failed: Type F must not exceed the size of the Callback class
  640 |         static_assert(sizeof(F) <= sizeof(Store) && alignof(F) <= alignof(Store),
      |                       ~~~~~~~~~~^~~~~~~~~~~~~~~~
mbed-os/platform/include/platform/Callback.h:640:33: note: the comparison reduces to '(16 <= 12)'
```

showing you that your lambda is 16 bytes, over the maximum of 12 bytes (3 words). To avoid this error, it is recommended to capture no more than three variables into the lambda, and to always capture by reference or pointer (unless you are only capturing values smaller than a pointer). Luckily, this error is checked for at compile time, so if your code compiles, you are OK. 

Also note that referencing member variables of an object only captures the pointer to the outer object, so if you do need lots of captures in a lambda you can use a struct. The following compiles OK:

```cpp
struct MyFlags {
    volatile bool flag1, flag2, flag3, flag4;
};
MyFlags flags;

auto nowThisFits = [&](uint8_t code, char const * message) {
    return flags.flag1 && flags.flag2 && flags.flag3 && flags.flag4;
};
EventHandler handler(nowThisFits);
```

## Empty Callbacks

Callbacks can also be empty, meaning they were created without a function to wrap:

```cpp
Callback<int(float)> cb1; // empty
Callback<int(float)> cb2(nullptr); // also empty
```

To check if a callback is empty, you should evaluate it as a boolean, or compare against nullptr.
```cpp
if(!cb1){} // will execute
if(cb2 == nullptr) // will also execute
```

## Currying with Lambdas

Even though callbacks do not directly support parameter binding / currying, you can achieve something similar through the use of lambdas. Extending the above event handler example, suppose you wanted to attach an event handler callback that took an event ID and an object to operate on, rather than the event ID and the message:

```cpp
class SomeOtherClass{...};
SomeOtherClass otherInstance;

void passEventToOtherClass(SomeOtherClass & instance, uint8_t code) {
    ...
}
```

You could do this with a lambda like so: 

```cpp
auto handler = [&](uint8_t code, char const * message) { 
    return passEventToOtherClass(otherInstance, code); 
};
addEventHandler(mbed::callback(handler));
```

## Worked Example

Suppose you have an ADC class which reads data from hardware, and you want that data to be passed to a low-pass filter each time a new sample is available. There are multiple ADCs and multiple low-pass filters, so you cannot use global functions. This could be implemented in the following manner:

``` cpp
class ADC {
    // Called when new data is generated by the hardware
    Callback<void(float)> newDataCallback{};
    
public:
    // In this example, the ADC read function calls the user-provided callback
    // when data is available.
    void attach(Callback<void(float)> const & cb) {
        // Assign the callback.
        // Note: In a real application you may need a mutex / critical section here 
        // if you want thread safety!
        newDataCallback = cb;
    }
};

class LowPass {
   float result;

public:
    // Move the low pass filter implementation to the ADC module
    void step(float data) {
        result = result*0.99 + data*0.01;
    }
};


// Our two adc modules
ADC adc1;
ADC adc2;

// Our two low-pass filters
LowPass low_pass1;
LowPass low_pass2;

int main() {
    adc1.attach(callback(&low_pass1, &LowPass::step));
    adc2.attach(callback(&low_pass2, &LowPass::step));
}
```


## Configuration

Two mbed_app.json configuration options permit trade-offs between image size and flexibility of the Callback class.

* `platform.callback-nontrivial` controls whether Callbacks can store non-trivially-copyable function objects. Having this setting off saves significant code size, as it makes Callback itself trivially-copyable, so all Callback assignments and copies are simpler. Almost all users use Callback only with function pointers, member function pointers or lambdas with trivial captures, so this setting can almost always be set to false. A compile-time error will indicate that this setting needs to be set to true if any code attempts to assign a non-trivially-copyable object to a Callback.

* `platform.callback-comparable` controls whether two Callbacks can be compared to each other. The ability to support this comparison increases code size whenever a Callback is assigned, whether or not any such comparison occurs. Turning the option off removes the comparison operator and saves a little image size.
