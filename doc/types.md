## Wire types and constants

   This language is strongly types, that is every port and every wire must have a type and two wires/ports can be connected if and only if they have the same type. Currently the following types are defined:

   + Basic types
     - `logic`
     - `void`
   + Arrays
   + Struct

### logic

   In VHDL/Verilog every port/signal must have a type. Usually you would assign to a single wire connection a logical type that also handles a wide class of states other than `0` and `1`, for example: high impedance `Z`, uninitialized `U`, unknown/runtime error `X`, anything goes `-`. In VHDL you can use `std_logic`/`std_ulogic` type whereas in Verilag you can use `wire`. In this language such type is called `logic` like in Verilog.

### array

   You can define array types with the following syntax:

   [*length*] *type*

   if you omit *type* then it is assumed `logic`. Moreover, *type* can itself be another array

       ports : [8]logic;    //declares 8 logic ports and threat them as a single port with eight entries numbered from 0 to 7
       portt : [8];    //same
       portu : [2][7];    // an array with length 2 of arrays of logics with length 7

   Indexes here are reversed in type definition in order to avoid C-multidimensional array reverse indices issue:

       port : [2][7] logic;

       port[0][1]    //ok
       port[1][5]    //ok
       // port[4][3]    Error: array port out of bounds, length=2 index=4
       // port[0][9]    Error: array port[0] out of bounds, length=7 index=9

### void
The last basic type is `void`. This type is very strange because it has only one possible value, which is `""`. Since it doen't hold or bring any information signals and ports of such type are never synthetized and can be safetly ignored when the component is instantiated. However, this type is useful in metaprogramming due to the following conversion rules:
   + `[N] void` is equal to `void` for every `N >= 0`;
   + `[0] ty` is equal to `void` for any type `ty`;
   + if `M < N` then `port[M:N]` and `port[N::M]` are equal to `""`.

### struct

   Structs are like structs in C, in particular they first have to be declared before use:

       struct A {
         porta : [2],
         portb : [3][4],
         portc,    //type logic
       }

       portc : A   //now portc has type A
       portc.porta   //this has type [2]

    Structs are used primarly to declare components as we will see in next section. Arrays of structures have an interesting behaviour: let consider a signal `a` with type `[2]A` where `A` is the struct defined before. Then you can see `a` both as an array of structures so `a[0]` is legal with type `A` and `a[0].portc` has type `logic` and as a new structure similar to `A` buth with fields doubled so `a.portc` is still valid but with type `[2]logic` now. In general for every index `i` and every field `fld` of a struct `ST` then the following equivalence holds fhr every signal `sig` of type `[N]ST`:

     sig[i].fld == sig.fld[i]

    The same holds for arrays of arrays of structs and similar.

[Prev](intro.md) [Next](comp.md)
