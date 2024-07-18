# Documentation

## Why a new HDL language?
   
There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

**EDIT:** I've taken a look at Lucid, which have a very similar syntax to this language and tries to achieve the same goal but it has already been adopted by some FPGA manifacturers. However, some things are handled differently, for example synchronized signals.

## Components

A `component` represents a digital component. Every component has a single input and a single output. Two components can be _connected_ by linking the output of one component to the input of the other one. The output of one component can be connected to the impunt of many different other components, but two (or more) outputs can not be connected to the same input. In other words, the input of every component is connected to exactly one output of another component.

Moreover, a component cannot have its output connected to its input, and the directed graph representing connections between components must be a directed acyclic graph.

## Types

Every input and output of a component must have a `type`. An input can be connected to an output only if both have the same type. There are three different classes of types:

+ Builtin types;
+ Component types;
+ Custom types;
+ Arrays.

### Builtin types
These types are defined internally by the compiler and can be used to define more complex types. Currently, the following builtin types are defined:

+ `logic`: a digital logic element that can hold two possible states, zero (`0`) and one (`1`). A `logic` connection is synthesized as a single digital pin.
+ `void`: an element that has only one possible state (the null state). Since its state is always defined and can not be changed the `void` type doesn't bring any information and can be used to disable entries. A component with `void` input doesn't need to be connected to any output and is synthetized as a component without input pins. Instead, a component with `void` doesn't provide any useful information to the outside world and so will never be synthetized to real hardware.
+ `!` (never type): an element that doesn't have any valid state and so can never be initialized. The only possible way to use the `!` type is inside a variant component in order to disable some of its entries.

### Component types
Components itself have types which nearly resemble functional types in languages like Haskell. Component types can also be used as input/output types of other components, and in this way you can define components with multiple input parameters.

There are two kinds of component types: _unnamed_ and _named_. An unnamed component type has the following signature:

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
These types are defined by users with the `record`, `variant` and `enum` constructs. These constructs also instantiate suitable components that allow you to instantiate them and retrieve informations. 

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


