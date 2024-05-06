## Wire types and constants

This language is strongly types, that is every port and every signal must have a type and two wires/ports can be connected if and only if they have the same type. Currently the following types are defined:

+ Root types
    - `logic`
    - `void`
    - structs
+ Arrays

Each root type also defines zero or more *fields* which are accessible to each port/signal with such type by using the `.` operator. For example if type `T` has the field `fld` of type `S` then for each signal `t` of type `T` you can access such field with the expression `t.fld` which has type `S`.

Each root type introduces

### logic

In VHDL/Verilog every port/signal must have a type. Usually you would assign to a single wire connection a logical type that also handles a wide class of states other than `0` and `1`, for example: high impedance `Z`, uninitialized `U`, unknown/runtime error `X`, anything goes `-`. In VHDL you can use `std_logic`/`std_ulogic` type whereas in Verilag you can use `wire`. In this language such type is called `logic` like in Verilog.

The `logic` type automatically defines the field `not` which returns the logical negation:

    x = `0;     //type logic, value 0
    y = x.not;  //y is now equal to 1


### void

This type is very strange because it has only one possible value, which is `""`. Since it doen't hold or bring any information signals and ports of such type are never synthetized and can be safetly ignored when the component is instantiated. 

### struct

Structs are like structs in C, in particular they first have to be declared before use:

    struct A {
        porta : [2],
        portb : [3][4],
        portc,    //type logic
    }

    portc : A   //now portc has type A
    portc.porta   //this has type [2]

Structs are used primarly to declare components as we will see in next section. 
### array

You can define array types with the following syntax:

    [*length*] *base type*

where *base type* can be any valid type (also another array). If you omit *base type* then it is assumed `logic`. However, sometimes the preceding expression doesn't generate an array type:

+ `[N] void` is equal to `void` for every `N >= 0`;
+ `[0] ty` is equal to `void` for any type `ty`;

Examples:

    ports : [8]logic;    //declares 8 logic ports and threat them as a single port with eight entries numbered from 0 to 
    portt : [8];    //same
    portu : [2][7];    // an array with length 2 of arrays of logics with length 7

You can access each element of an array signal/port with square braces as in C language. However, indices here are reversed in type definition in order to avoid C-multidimensional array reverse indices issue:

    port : [2][7] logic;
    port[0][1]    //ok
    port[1][5]    //ok
    // port[4][3]    Error: array port out of bounds, length=2 index=4
    // port[0][9]    Error: array port[0] out of bounds, length=7 index=9

Arrays don't introduce any new field, however they inherit all the fields belonging to their base type. Let consider a signal `a` with type `[2]A` where `A` is a struct with the field `portc`. Then you can see `a` both as an array of structures, so `a[0]` is legal with type `A` and `a[0].portc` has type `logic`, and as a new structure similar to `A` buth with fields doubled so `a.portc` is still valid but with type `[2]logic` now. In general for every index `i` and every field `fld` of a struct `ST` then the following equivalence holds fhr every signal `sig` of type `[N]ST`:

    sig[i].fld == sig.fld[i]

The same holds for arrays of arrays and so on.

[Prev](intro.md) [Next](comp.md)
