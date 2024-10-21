# Documentation

## Why a new HDL language?
   
There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

## Components

The primary elements in this languages are called *components*. Every component has always an *output port* and an *input port*, both of them must have a *type*. Components can be *linked* to form more complex components. More preciselt, a component `A` can be linked to component `B` if and only if the output port type of `A` is equal to the input port type of `B`. 

Every component must first be *instantiated* from a *design* in order to be used.

A design for a component can be defined in the following way:

<pre><code>design <i>design_name</i>[<i>input_type</i>]  = <i>component chain</i>;</code></pre>

where *input_type* is the type of the input port and *component chain* is a sequence of linked components that describe the behaviour of your custom component. You don't need to specify also the output port type because it will be inferred by *input_type* and *component chain*.

If you omit <code>[<i>input_type</i>]</code> then `void` is inferred.

## Component instantiation and linking

Inside a component chain you can instantiate other components by using the symbol `.` followed by the design name of the component. For example `.adder` instantiate a new component by using the design named `adder`.

Two or more components can be linked (or *composed*) by using one or more spaces between them. For example let `inc` be the design of a component that increases its input (seen as an unsigned integer) by 1 and `dup` instead duplicates its input, then the following component design

    design comp[num] = .dup .inc;

would compute 2x+1 from input x. Instead,

    design comp2[num] = .inc .dup;

would compute 2(x+1).

## Builtin types

There are three different classes of types:

+ Builtin types;
+ Structures and arrays;
+ Unions.

Builtin types are defined internally by the compiler and can be used to define more complex types. All these types are copyable. Currently, the following builtin types are defined:

+ `logic`: a digital logic element that can hold two possible states, zero (`logic::0`) and one (`logic::1`). A `logic` connection is synthesized as a single digital pin.
+ `void`: an element that has only one possible state (the null state). Since its state is always defined and can not be changed the `void` type doesn't bring any information and can be used to disable entries. A component with `void` input doesn't need to be connected to any output and is synthetized as a component without input pins. Instead, a component with `void` doesn't provide any useful information to the outside world and so will never be synthetized to real hardware.
+ `!` (never type): an element that doesn't have any valid state and so can never be initialized. The only possible way to use the `!` type is inside a variant component in order to disable some of its entries.

*Remark*: A component with input port of type `void` is called *element*. The type of an element is always the type of its output port since the input port type must be `void`. Examples of elements are `b0` and `b1` which have both type `logic` and represent zero and one logical statuses respectively. 


## Custom types
Custom types are defined with the following general syntax:

<pre><code>type <i>newtype</i> = <i>...</i>;</code></pre>

where *newtype* is your custom type name. Type and components live in the same "namespaces" therefore you will have a name clash when you try to define a new type with the same name of an already existent component (however, you can define anonymous types within components).

### Structures

A structure type can be declared in the following way:

<pre><code>type <i>newtype</i> = struct {
  <i>field1</i> : <i>type1</i>,
  <i>field2</i> : <i>type2</i>,
   ...
  <i>fieldn</i> : <i>typen</i>,
}</code></pre>

just like structures in C. You can omit the `: typei` part for one or more fields, in that case `typei` is assumed to be `logic`. Moreover, each field *fieldi* is also a component design with input port type the struct itself and output port type the respective *typei*.

When you define a new structure type `newtype` then for each field `fieldi` of `newtype` with type `typei` a new designer named `newtype::fieldi` that retrieves the selected field from the input struct. Therefore, its input port type would be `newtype` and its output port type `typei`, the type of `fieldi`.

For example, we consider the following struct type:

    type ST = struct {
      field_a : A,
      field_b : B,
    }

then the following component design

    design get_a [ST] = .ST::field_a;

has output port type `ST` and just retrieves the content of field `field_a` from the input struct.

#### Struct element

Let `type adder_ty = struct {add1 : num, add2 : num};` and `adder` be a design of a component with input type `adder_ty` and output type `num` which returns the sum of `add1` and `add2`. In order to use `adder` in your component chain you can do one of the following things:

- set `adder_ty` as your component input type and then pass it directly to `.adder`;
- instantiate another component with `adder_ty` as output port type and link it to `adder`;
- instantiate a *struct element* and link it to `adder`.

As its name says, a struct element is a component with input port type `void` and the selected struct as the output port type with their fields initialized with component chains:

<pre><code>$adder_ty{
   add1 = <i>chain1</i>;
   add2 = <i>chain2</i>;
} .adder</code></pre>

where *chain1* and *chain2* are component chains describing elements (so with input port type `void`) with output port type tyhe type of *add1* and *add2* respectively (which is `num` for both of them).

If you don't want to initialize a field from a new component chain but you want instead link it to a component output port then you can specify the chosen field in parenthesis:

<pre><code>$adder_ty (add1){
   add2 = <i>chain2</i>;
} .adder</code></pre>

Now `$adder_ty(add1)` is no more an element since its input port type is `num` (the type of `add1`) since it uses the input port to initialize the field `add1`. Field `add2` is still initialized with the component chain `chain2`.

Once you have the `adder` component design, you can define the component `inc` (which increment the input number by 1) in the following way:

    design inc[num] = $adder_ty(add1) {add2 = .num_1; } .adder;

where `num_1` is an element which returns 1 as a `num`.

### Unions

An union type can be declared in the following way:

<pre><code>union {
  <i>field1</i> : <i>type1</i>,
  <i>field2</i> : <i>type2</i>,
   ...
  <i>fieldn</i> : <i>typen</i>,
}</code></pre>

You can omit the `: typei` part for one or more fields, in that case `typei` is assumed to be `void`. Moreover, each field *fieldi* is also a component design with input port type *typei* and output port type the union itself.

When you define a new union type `newtype` then for each field `fieldi` of `newtype` with type `typei` a new designer named `newtype::fieldi` that generates the union from the selected field. Therefore, its input port type would be `typei`, the type of `fieldi`, and its output port type `newtype`.

For example, we consider the following union type:

    type UN = union {
      field_a : A,
      field_b : B,
    }

then the following component design

    design get_a [A] = .UN::field_a;

has output port type `A` and returns the union generated by using field `field_a`.

The `logic` builtin type can be seen as an union defined in the following way:

    type logic = union {
      0,
      1,
    }

#### Match component

### Anonymous types

## Inline components
May happen that you component can't be representable as only composition of already existent components, and you need to use your input in multiple places of your component chain. In such cases you can declare *inline components* inside a component chain in the following way:

<pre><code>|<i>param</i>| <i>component chain</i> </code></pre>

where *component chain* describes a component with `void` as input port type and you can use *param* inside it as an instantiable component in order to use the input value in different places.

For example, then we can implement the duplicating component `dup` by using `adder` in the following way:

    design dup[num] = |x| $adder {add1 = .x; add2 = .x;};

Moreover, you can use the expression

<pre><code>save(<i>x</i>)</code></pre>

as a synonym of

<pre><code>|<i>x</i>| .<i>x</i></code></pre>
