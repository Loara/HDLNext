# Documentation

## Why a new HDL language?
   
There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

## Components

The primary elements in this languages are called *components*. Every component has always an *output port* and an *input port*, both of them must have a *type*. Components can be *composed* to form more complex components.

Every component must first be *instantiated* from a *design* in order to be used.

A design for a component can be defined in the following way:

<pre><code>design <i>design_name</i>[<i>input_type</i>]  = <i>expression</i>;</code></pre>

where *input_type* is the type of the input port and *expression* describes the internal structure of your component. You don't need to specify also the output port type because it will be inferred by *input_type* and *expression*.

If you omit <code>[<i>input_type</i>]</code> then `void` is inferred.

## Component instantiation and linking

Inside an expression you can instantiate other components by using the symbol `\` followed by the design name of the component. For example `\adder` instantiate a new component by using the design named `adder`.

Two or more components can be *linked* (or *composed*) by using one or more spaces between them. For example let `inc` be the design of a component that increases its input (seen as an unsigned integer) by 1 and `dup` instead duplicates its input, then the following component design

    design comp[num] = \dup \inc;

would compute 2x+1 from input x. Instead,

    design comp2[num] = \inc \dup;

would compute 2(x+1).

## Types

There are three different classes of types:

+ Builtin types;
+ Structures and arrays;
+ Unions;
+ Custom types.

### Builtin types
These types are defined internally by the compiler and can be used to define more complex types. All these types are copyable. Currently, the following builtin types are defined:

+ `logic`: a digital logic element that can hold two possible states, zero (`0`) and one (`1`). A `logic` connection is synthesized as a single digital pin.
+ `void`: an element that has only one possible state (the null state). Since its state is always defined and can not be changed the `void` type doesn't bring any information and can be used to disable entries. A component with `void` input doesn't need to be connected to any output and is synthetized as a component without input pins. Instead, a component with `void` doesn't provide any useful information to the outside world and so will never be synthetized to real hardware.
+ `!` (never type): an element that doesn't have any valid state and so can never be initialized. The only possible way to use the `!` type is inside a variant component in order to disable some of its entries.

*Remark*: A component with input port of type `void` is called *element*. The type of an element is always the type of its output port since the input port type must be `void`. Examples of elements are `b0` and `b1` which have both type `logic` and represent zero and one logical statuses respectively. 

### Structures and array

A structure type can be declared in the following way:

<pre><code>struct {
  <i>field1</i> : <i>type1</i>,
  <i>field2</i> : <i>type2</i>,
   ...
  <i>fieldn</i> : <i>typen</i>,
}</code></pre>

just like structures in C. You can omit the `: typei` part for one or more fields, in that case `typei` is assumed to be `logic`. Moreover, each field *fieldi* is also a component design with input port type the struct itself and output port type the respective *typei*.

An *array* is a particular structure where each *fieldi* is equal to *i* and are declared in the following way:

<pre><code>[<i>n</i>]<i>type</i></code></pre>

which is equivalent to

<pre><code>struct {
  0 : <i>type</i>,
  1 : <i>type</i>,
  2 : <i>type</i>,
   ...
  <i>n-1</i> : <i>type</i>,
}</code></pre>

Two structure types are the same type if and only if have the same fields with the same name and types, ordering doesn't matter. To distinguish equal structures you should wrap them in *custom types*.

### Unions

An union type can be declared in the following way:

<pre><code>union {
  <i>field1</i> : <i>type1</i>,
  <i>field2</i> : <i>type2</i>,
   ...
  <i>fieldn</i> : <i>typen</i>,
}</code></pre>

You can omit the `: typei` part for one or more fields, in that case `typei` is assumed to be `void`. Moreover, each field *fieldi* is also a component design with input port type *typei* and output port type the union itself.

Two union types are the same type if and only if have the same fields with the same name and types, ordering doesn't matter. To distinguish equal unions you should wrap them in *custom types*.

### Custom types
A custom type is a type that is created by users by using already existent types. To define a new type *newtype* from an already existent type *oldtype* you can use the following declaration:

<pre><code>type <i>newtype</i> = <i>oldtype</i>;</code></pre>

Notice that *newtype* is different from *oldtype*, and two custom types are the same type if and only if they have the same name. Moreover, *oldtype* is implicitly convertible to *newtype* (but not the converse) and *newtype* can also be used as a component design with input port of type *oldtype* and output port type *newtype*.

## Struct components
Struct components are special components that have structs as output port type. Struct componenrs can be *simple* or *linked*. A simple struct component can be instantiated in the following way:

<pre><code>struct{
  <i>field1</i> = <i>exp1</i>;
  <i>field2</i> = <i>exp2</i>;
  <i>field3</i> = <i>exp3</i>;
    ...
  <i>fieldn</i> = <i>expn</i>;
}</code></pre>

where each *expi* is an expression of an element with the same type of field *fieldi*. This expression instantiates an element with (output) type the struct with fields *field1*, *field2*, ..., *fieldn*.

For example let `adder` be a component with input port type `struct {add1 : num, add2 : num}` which returns add1 + add2, and let `num3`, `num4` be two elements with type `num` returning 3 and 4 respectively. Then an element `num7` returning 7 can be defined in the following way:

    design num7 =  struct {add1 = \num3; add2 = \num4;} \adder;

Instead a linked struct component is declared in the following way:

<pre><code>struct(<i>fieldz</i>){
  <i>field1</i> = <i>exp1</i>;
  <i>field2</i> = <i>exp2</i>;
  <i>field3</i> = <i>exp3</i>;
    ...
  <i>fieldn</i> = <i>expn</i>;
}</code></pre>

where *fieldz* is another field of the struct different from others *fieldi* fields which value is not set from an assigned expression. Instead, is binded to the instantiated component input port.

For example, a component `inc3` which takes a `num` as input and returns its value increased by 3 can be defined in the following way:

    design inc3[num] = struct(add1) {add2 = \num3;} \adder;

where the value passed to `inc3` as input is directly assigned to `add1` field and then passed to `adder`. 

## Inline components
May happen that you component can't be representable as only composition of already existent components, and you need to use your input in multiple places of your expression. In such cases you can declare *inline components* in the following way:

<pre><code>|<i>param</i>| {<i>expression</i>} </code></pre>

where *expression* is an expression describing an element (a component with `void` as input type port) in which you can use *param* as an element design. The resulting component would then have *ptype* as input port type and the *expression* type as output port type.

For example, then we can implement the duplicating component `dup` by using `adder` in the following way:

    design dup[num] = |x| { struct {add1 = \x; add2 = \x;} \adder };

