1. Why a new HDL language?
   
   There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

   However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

2. Wire types and constants
   
   In VHDL/Verilog every port/signal must have a type. Usually you would assign to a single wire connection a logical type that also handles a wide class of states other than `0` and `1`, for example: high impedance `Z`, uninitialized `U`, unknown/runtime error `X`, anything goes `-`. In VHDL you can use `std_logic`/`std_ulogic` type whereas in Verilag you can use `wire`. In this language such type is called `wire` like in Verilag, and every port/signal without a type is implicitly of type `wire`.

       in port1; //by default it has type wire

   Constants of type `wire` should be preceded by ` symbol:

       port1 = `1;    //assigns logic state 1 to port1
       port2 = `Z;     //assign high impedance state to port2

   You can define array types with the following syntax:

   [*length*] *type*

   if you omit *type* then it is assumed `wire`. Moreover, *type* can itself be another array

       out ports : [8]wire;    //declares 8 std_ulogic ports and threat them as a single port of "type" std_ulogic[8] numbered from 0 to 7
       out portt : [8];    //same
       in portu : [2][7];    // an array with length 2 of arrays of wires with length 7

   Indexes here are reversed in type definition in order to avoid C-multidimensional array reverse indices issue:

       in port : [2][7] wire;

       port[0][1]    //ok
       port[1][5]    //ok
       // port[4][3]    Error: array port out of bounds, length=2 index=4
       // port[0][9]    Error: array port[0] out of bounds, length=7 index=9

   You can access each element by specifying its index

       porta = ports[1];    //set porta std_ulogic signal to the element of ports with index 1

    or you can access a continuous range of elements with the : operator

       portt[0:2] = ports[4:6];    //equivalent to portt[0]=ports[4]; portt[1]=ports[5]; portt[2]=ports[6];

   Arrays can be accessed in reverse order with the `rev` unary operator:

       portt[0:2] = rev(portr[0:2]);    //portt[0] = portr[2]; portt[1] = portr[1]; portt[2] = portr[0];

   Array constants must be defined between " symbols

       porttt[0:2] = "01U";    //equivalent to portt[0]=`0; portt[1]=`1; portt[2]=`U;

   you can assign integer constants to array ports/signals by using one of the following prefixes: `0d` (decimal), `0x` (hexadecimal), `0o` (octal)

       portu[0:2] = 0d3;    //equivalent to portu[0] = `1; portu[1] = `1; portu[2] = '0
       //portu[0:1] = 0d6; ERROR: 6 needs at least 3 bits

4. Components and implementations
5. Signals and parallel coding
6. sync signals and sequential coding
7. Automatic code generation with variables, `static if` and `static repeat`
8. Template parameters
