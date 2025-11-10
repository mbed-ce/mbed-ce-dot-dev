# Callback

A callback is a user provided function that a user may pass to an API. The callback allows the API to execute the user’s code in its own context.

The Callback class manages C/C++ function pointers so you don't have to. If you are asking yourself why you should use the Callback class, you should read the [Importance of State](platform-concepts.html#the-importance-of-state) section.

#### Why should you use Callbacks?

Supporting all of the standard C++ function types is difficult for an API developer. An API developer must consider state, C++ Function objects, const correctness and volatile correctness.

State is important, so an API developer must support either C-style function pointers with state, or C++ member function pointers. Stateless callbacks are just as common, but passing a stateless callback as a member function requires writing a lot of boilerplate code and instantiating an empty class. Further, an API developer also must support a standard function pointer.

Another common design pattern is the function object, a class that overrides the function call operator. A user may pass function objects as C++ member function pointers and C++ requires a large set of overloads to support all of the standard function types. It is unreasonable to expect a new library author to add all of these overloads to every function that could take in a callback.

A useful C++ feature is compile time const-correctness checks, which increases API complexity when combined by callbacks with state. To allow a user to take full advantage of the const-correctness checks, a C++ API must support both the const and non-const versions of member function pointers.

Another useful C++ feature is volatile-correctness. When volatile-correctness is necessary, we expect that the user hides volatile members inside of a non-volatile class.

C++ provides the tools to delegate this complexity to a single class. This class is the Callback class. The Callback class should be familiar to users of the std::function class that C++11 introduced and is available for older versions of C++.

<h6 id="the-importance-of-state">The importance of state</h6>

Callbacks may have two important pieces of information, the code to execute and the state associated with the callback.

A common API design mistake is to use a callback type that doesn’t allow a user to attach state to a callback. The most common example of this is a simple C function pointer:

```c++
class ADC {
public:
    // Here, the adc_callback_t type is a function that takes in data
    typedef void (*adc_callback_t)(float data);


    // In this example, the ADC read function calls the user-provided callback
    // when data is available.
    void attach(adc_callback_t cb);
};
```

This API is sufficient for simple applications, but falls apart when there are multiple ADC modules available. This problem becomes especially noticeable when a user tries to reuse the same procedure for multiple callbacks.

For example, consider applying a low-pass filter to two different ADC modules:

``` c++ TODO
// Here is a small running-average low-pass filter.
float low_pass_result;
void low_pass_step(float data) {
     low_pass_result = low_pass_result*0.99 + data*0.01;
}

// Our two adc modules
ADC adc1;
ADC adc2;

int main() {
    adc1.attach(low_pass_step);

    // Problem! Now both low pass filters share the same state!
    adc2.attach(low_pass_step);
}
```

Without state, callbacks compose poorly. In C, you fix this by adding a "state" argument to the function pointer, and by passing opaque "state" when you register the callback.

Here’s the low-pass example using an additional argument for state.

```c++ TODO
class ADC {
   public:
   // Here, the adc_callback_t type is a function that takes in data, as well as a pointer for state
    template <typename T>
    typedef void (*adc_callback_t)(T *state, float data);


    // In this example, the ADC read function calls the user-provided callback
    // when data is available.
    template <typename T>
    void attach(adc_callback_t<T> cb, T *state);
};

// Here is a small running-average low-pass filter.
void low_pass_step(float *result, float data) {
     *result = *result*0.99 + data*0.01;
}

// Our two adc modules
ADC adc1;
ADC adc2;

// Our two low-pass filter results
float low_pass_result1;
float low_pass_result2;

int main() {
    adc1.attach(low_pass_step, &low_pass_result1);

    // Register a second low pass filter, no more issues!
    adc2.attach(low_pass_step, &low_pass_result2);
}
```

One of the core features of C++ is the encapsulation of this "state" in classes, with operations that modify the state being represented as member functions in the class. Member function pointers are not compatible with standard function pointers. The Callback class allows API authors to implement a single interface that accepts a Callback, and the user may provide a C function and state or C++ member function and object without special consideration by the API author.

Here’s the low-pass filter example rewritten to use the callback class:

``` c++ TODO
class ADC {
public:
    // In this example, the ADC read function calls the user-provided callback
    // when data is available.
    void attach(Callback<void(float)> cb);
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


## Calling callbacks

Callbacks overload the function call operator, so you can call a Callback like you would a normal function:

```c++
void run_timer_event(Callback<void(float)> on_timer) {
    on_timer(1.0f);
}
```

The only thing to watch out for is that the Callback type has an empty Callback, just like a null function pointer. Default initialized callbacks are empty and assert if you call them. If a callback may be empty, you need to check if it is empty before calling it.

``` c++
void run_timer_event(Callback<void(float)> on_timer) {
    if (on_timer) {
        on_timer(1.0f);
    }
}
```

You can reset Callbacks to empty by assigning `nullptr`.

The Callback class is what’s known in C++ as a “Concrete Type”. That is, the Callback class is lightweight enough to be passed around like an int, pointer or other primitive type.

## Configuration

Two system configuration options permit trade-offs between image size and flexibility of the Callback class.

* `platform.callback-nontrivial` controls whether Callbacks can store non-trivially-copyable function objects. Having this setting off saves significant code size, as it makes Callback itself trivially-copyable, so all Callback assignments and copies are simpler. Almost all users use Callback only with function pointers, member function pointers or lambdas with trivial captures, so this setting can almost always be set to false. A compile-time error will indicate that this setting needs to be set to true if any code attempts to assign a non-trivially-copyable object to a Callback.

* `platform.callback-comparable` controls whether two Callbacks can be compared to each other. The ability to support this comparison increases code size whenever a Callback is assigned, whether or not any such comparison occurs. Turning the option off removes the comparison operator and saves a little image size.

<span class="tips">**Tip:** See the documentation of the [Arm Mbed configuration system](../program-setup/advanced-configuration.html) for more details about `mbed_app.json`. </span>

## Callback class reference

[![View code](https://www.mbed.com/embed/?type=library)](https://os.mbed.com/docs/mbed-os/development/mbed-os-api-doxy/classmbed_1_1_callback.html)

## Serial passthrough example with callbacks
[![View code](https://www.mbed.com/embed/?url=https://github.com/ARMmbed/mbed-os-snippet-Callback_SerialPassthrough/tree/v6.7)](https://github.com/ARMmbed/mbed-os-snippet-Callback_SerialPassthrough/blob/v6.7/main.cpp)

## Thread example with callbacks

The Callback API provides a convenient way to pass arguments to spawned threads. This example uses a C function pointer in the Callback.

 [![View code](https://www.mbed.com/embed/?url=https://github.com/ARMmbed/mbed-os-snippet-Threading_with_callback/tree/v6.7)](https://github.com/ARMmbed/mbed-os-snippet-Threading_with_callback/blob/v6.7/main.cpp)

## Sonar example

Here is an example that uses everything discussed in the [introduction to callbacks](../apis/platform-concepts.html#callbacks) document in the form of a minimal Sonar class. This example uses a C++ class and method in the Callback.

[![View code](https://www.mbed.com/embed/?url=https://github.com/ARMmbed/mbed-os-snippet-Sonar/tree/v6.7)](https://github.com/ARMmbed/mbed-os-snippet-Sonar/blob/v6.7/main.cpp)
