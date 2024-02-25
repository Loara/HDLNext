1. Why a new HDL language?
   
   There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

   However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

   **EDIT:** I've taken a look at Lucid, which have a very similar syntax to this language and tries to achieve the same goal but it has already been adopted by some FPGA manifacturers. However, some things are handled differently, for example synchronized signals.

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

       portt = ports[4:6];    //equivalent to portt[0]=ports[4]; portt[1]=ports[5]; portt[2]=ports[6];

   Arrays can be accessed in reverse order with the `rev` unary operator:

       portt = rev(portr[0:2]);    //portt[0] = portr[2]; portt[1] = portr[1]; portt[2] = portr[0];

   Array values can be declared with `[`, `]` brackets:

       porta = [`0, `1, `Z];    //porta[0] = `0; porta[1] = `1; porta[2] = `Z;

   moreover, wire array constants can be also declared between `"` symbols: 

       porttt = "01U";    //equivalent to portt[0]=`0; portt[1]=`1; portt[2]=`U;

   Integer constants can be assigned to wire arrays. Integers have the following format:

   *<num bits>*_<radix>_*<number>*

   where *<radix>* can be `d` (decimal), `h` (hexadecimal) or `o` (octal). Less significant bits are assigned to lower indices first:

       porta = 3d3;    //porta[0] = `1; porta[1] = `1; porta[2] = `0;

3. Components and implementations

   A component in this language can be define in the following way: 
   <pre><code>
   comp <i>name</i> {
     in {
       //input ports
     }
     out {
       //output ports
     }
     impl {
       //implementation
     }
   }
   </code></pre>
   which is very different from VHDL/Verilog because they force you to separate the model and the implementation. However, you can achieve the same outcome by using `design`:

   <pre><code>
   design <i>des_name</i> {
     in {
       //input ports
     }
     out {
       //output ports
     }
   }
   </code></pre>

   which can be later associated to one or more effective implementations:
   
   <pre><code>
   comp <i>name</i> : <i>des_name</i> {
     impl {
       //implementation
     }
   }
   </code></pre>

   Designs are also useful for templates/generics.

   Ports must be defined in `in` (for input ports) and `out` (for output ports) subblocks in `comp` or `design` blocks (*TODO*: introduce `inout` ports). A port definition follows this syntax:

   <pre><code>
      <i>port_name</i> : <i>type_name</i>;
   </code></pre>

   If you don't specify *type_name* then `wire` is assumed. Example:

       design multiplex_4_1 {
         in {
           adr : [2]wire;
           input : [4];
         }
         out {
           output;    //assumed wire
         }
       }
   
4. Signals and parallel coding

   Inside the `impl` subblock you put all the logical ports, connections and components you need in order to implement the behaviour of your component. This block consists in a list of assigments on the following form:

     *signal/out port name* = *signal expression*;

   All these statements are (theoretically) executed in parallel so ordering doesn't matter. In VHDL/Verilog you must first define all your signals and declare their type before using it. Here signals aren't declared in a separate line, but they are defined when you use it for the first time. However, each signal/ouput port can appear at the left side of `=` symbol **only once**, so the following code would issue an error

       A = `1;
       A = `0;

   but the following one is correct

       A = B;
       B = `1;

   even if `B` was not declared before `A` declaration because ordering doesn't matter, moreover at the end of execution both `A` and `B` would be at state `1` because here execution is parallel.

   Remember that in an `impl` block every output port must be assigned exactly once since all the output ports must have a value.

   In a signal expression you can use other signals, input and output ports, constants and operators. You can use parenthesis to control ordering of operations. Actually the following operations are defined:

   - Logical `not`, `and`, `or`, `nand`, `nor`, `xor`, `xnor`:

         ...
         A = (not B) and C and (D xor `1);
         ...

     these operators works both on single `wire` types, arrays of `wire`, array of array of `wire` and so on, by applying the chosen operation on each element. However, both the arguments must have the same type because this language is strongly typed.
   - Range operator `[min:max]` as described before;
   - Array composition `[...]`:

         A = `0;  //type wire
         B = `1;  //ditto
         C = [A, B]; //equal to "01", type [2]wire
         D = [A];  //type [1]wire, which is different from wire
     
   - Array concatenation `~`:

         A = "0";  //type [1]wire
         B = "1";  //type [1]wire
         C = A ~ B;  //equal to "01", type [2]wire
  
   - Component allocation `alloc`. This operator is a bit complex and needs first an example. Suppose we have already defined the following adder component:

         comp ADDER {
           in {
             a;
             b;
             in_rem;
           }
           out {
             sum;
             out_rem;
           }
           impl {
             ...
           }
         }

     and we want to instantiate it in another component. You can do it thanks to the `alloc` operator in the following way:

         adr = alloc ADDER {
             a = C;
             b = D;
             in_rem = `0;
           };

     where `C`, `D`, `E` are other signals with compatible types. Now `adr` is not a `wire` or array signals, so you can put it directly in other signal expression. To take its output you must use the `.` operator:

         sig = adr.sum;    //reads the sum port of ADDER
         sig_rep = not adr.out_rem;  //uses the out_rem port of ADDER

     if you need only one output value then wyou can apply the `.` operator directly on the valued returned by `alloc`:

         sig = alloc ADDER {
             a = C;
             b = D;
             in_rem = `0;
           }.sum;
     
5. `sync` signals and sequential coding

   Parallel coding defined in the previous part has a very restrictive requirement: no cyclic assigments. For example this code wouldn't compile

       A = B;
       B = A;

   because the value of `A` depends on `B` and vice versa. Cyclic connections are fundamental for sequential hardware, therefore VHDL/Verilog introduced `process` blocks that evaluated signal sequentially. This language instead uses a completely different approach by defining `sync` signals. A `sync` signal is defined in the following way:
   <pre><code>sync(<i>clock expression</i>) <i>signal_name</i> = <i>signal expression</i>;</code></pre>
   here *clock expression* is a regular signal expression that determine the "state" in which the defined signal stays. For example let `clock` be a input port/signal, then `sync(clock) A = ...` not only defines the signal `A`, but also defines the "virtual input port" `^A` that holds the values of `A` *in the previous state* determined by `clock`. When `clock` switch
9. Automatic code generation with variables, `static if` and `static repeat`
10. Template parameters
