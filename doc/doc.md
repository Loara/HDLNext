# Documentation

## Why a new HDL language?
   
There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

## Objects

The primary elements in this languages are called *objects*. Each object has a unique *type* and must be *instantiated* in order to be used. In order to instantiate a component you need a *designer* that informs the compiler on how to build the selected component. 

A very important class of objects are the so-called *components*. Each component has an *input* and an *output*, both of them have a type. Every component can be evaluated at a provided object with the same type of component input in order to provide a new object with the same type of component output one.

## Types

A type can be *copyable* or not. Every instanced object with a copyable type can be used multiple times, instead an object with a non copyable type can be used only once, if you need to use it again you have to instantiate a new object. There are three different classes of types:

+ Builtin types;
+ Component types;
+ Custom types;
+ Arrays.

### Builtin types
These types are defined internally by the compiler and can be used to define more complex types. All these types are copyable. Currently, the following builtin types are defined:

+ `logic`: a digital logic element that can hold two possible states, zero (`0`) and one (`1`). A `logic` connection is synthesized as a single digital pin.
+ `void`: an element that has only one possible state (the null state). Since its state is always defined and can not be changed the `void` type doesn't bring any information and can be used to disable entries. A component with `void` input doesn't need to be connected to any output and is synthetized as a component without input pins. Instead, a component with `void` doesn't provide any useful information to the outside world and so will never be synthetized to real hardware.
+ `!` (never type): an element that doesn't have any valid state and so can never be initialized. The only possible way to use the `!` type is inside a variant component in order to disable some of its entries.

### Component types
Components itself have types which nearly resemble functional types in languages like Haskell. Component types can also be used as input/output types of other components, and in this way you can define components with multiple input parameters. Component types are never copyable.

 are two kinds of component types: _unnamed_ and _named_. An unnamed component type has the following signature:

    in_type -> out_type

Examples:

    logic -> logic                      //both input and output are logic
    void -> logic                       //input is void and output is logic, functionally equivalent to logic
    logic -> (logic -> [2]logic)        //input is logic, output is another component with input of type logic and output an array of logic with length 2
    (logic -> logic) -> logic -> logic  //input is a component with both input and output logic, output is a component with both input and output logic
                                        //equivalent to (logic -> logic) -> (logic -> logic)

A named component type has instead the following signature

    #parameter in_type -> out_type

where `parameter` is an alfanumeric string representing the imput parameter. This is useful when you want to allow the user to discern each component parameter during instantiation. Every named component can always be implicitly converted to its unnamed version.

Remember that if you set a component type to an input then that imput parameter _can not be copied_: you can use it only once inside your component.

There are two fundamental components already defined by the compiler: `0` and `1`. Both have type `void -> logic` and always return respectively the low logical value and the high logical value.

### Custom types
A custom type is a type that is created by users by using already existent types. When you define a new custom type the compiler also generates different designers in order to make them usable in your code.

#### Basic type
You can define a new type _T_ which is logically equivalent to an already existent type _N_ with the `type` keyword:

    type T = N;

That also generates the following designers:

    T : comp N -> T;
    T::into : comp T -> N;

#### Record type
A record type works like structs in C and can be defined with the `record` keyword:

    record T {
        #par1 N,
        #par2 [2]logic,
        #par3,  //implicit logic
    };

and defines the following designers:

    T : comp (#par1 N) -> (#par2 [2]logic) -> (#par3 logic) -> T;
    T::par1 : T -> N;
    T::par2 : T -> [2]logic;
    T::par3 : T -> logic;

Moreover, you can use the dot operator `.` on an object `t` of type `T`

    t.par2

which is equivalent to

    \T::par2 t

#### Variant type
A variant type works like structs in C and can be defined with the `variant` keyword:

    record T {
        #var1 N,
        #var2 logic,
        #var3,  //implicit void
    };

and defines the following designers:

    T::var1 : N -> T;
    T::var2 : logic -> T;
    T::var3 : void -> T;    //equivalent to T

### Array
You can define array types with the following syntax:

    [length] type

where `type` can be any valid type (also another array) and `length` is a nonnegative integer. If you omit `type` then `logic` is inferred.

Examples:

    [8]logic        //declares 8 logic ports and threat them as a single port with eight entries numbered from 0 to 
    [8]             //same
    [2][7]          // an array with length 2 of arrays of logics with length 7
    [3](logic -> T) //an array with length 3 of components with logic input and custom type T output

You can access each element of an array signal/port with square braces as in C language. However, indices here are reversed in type definition in order to avoid C-multidimensional array reverse indices issue:

    port : [2][7] logic;
    port[0][1]    //ok
    port[1][5]    //ok
    // port[4][3]    Error: array port out of bounds, length=2 index=4
    // port[0][9]    Error: array port[0] out of bounds, length=7 index=9


## Expressions

A (named) component is declared in the following way:

    name : comp (#parameter : in_type) = expression;

where `name` is the component name, `parameter` is the input parameter name, `in_type` the input type and `expression` is an expression that describe the component behaviour. You don't need to specify the output type since it can always be inferred from `expression`, however you can explicitly specify it in order to avoid errors in this way:

    name : comp(#parameter : in_type) [out_type] = expression;


