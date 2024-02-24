1. Why a new HDL language?
   
   There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

   However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

2. Wire types and constants
   
   In VHDL every port/signal must have a type. Usually you would assign to a single port the type `std_logic` (or `std_ulogic`) because it handles a wide class of states other than `0` and `1`, for example: high impedance `Z`, uninitialized `U`, unknown/runtime error `X`, anything goes `-`. In this language every port/signal has by default the type `std_ulogic` (and not `std_logic` for reasons that will become clear later):

       in port1; //by default it has type std_ulogic

   Constants of type `std_ulogic` should be preceded by ` symbol:

       port1 = `1;    //assigns logic state 1 to port1
       port2 = `Z;     //assign high impedance state to port2

   You can define array types like arrays in C language:

       out ports[8];    //declares 8 std_ulogic ports and threat them as a single port of "type" std_ulogic[8] numbered from 0 to 7

   you can access each element by specifying its index

       porta = ports[1];    //set porta std_ulogic signal to the element of ports with index 1

    or you can access a continuous range of elements with the : operator

       portt[0:2] = ports[4:6];    //equivalent to portt[0]=ports[4]; portt[1]=ports[5]; portt[2]=ports[6];

   to reverse

   Array constants must be defined between " symbols

       porttt[0:2] = "01U";    //equivalent to portt[0]=`0; portt[1]=`1; portt[2]=`U;

   you can assign integer constants to array ports/signals by using one of the following prefixes: `0d` (decimal), `0x` (hexadecimal), `0o` (octal)

       portu[0:2] = 0d3;    //equivalent to portu[0] = `1; portu[1] = `1; portu[2] = '0
       //portu[0:1] = 0d6; ERROR: 6 needs at least 3 bits

3. Concurrent coding
   VHDL needs to explicitly each signal inside a component:

       signal A, B std_ulogic

       A <= 0;
       B <= A;
