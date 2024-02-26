1. Why a new HDL language?
   
   There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

   However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

   **EDIT:** I've taken a look at Lucid, which have a very similar syntax to this language and tries to achieve the same goal but it has already been adopted by some FPGA manifacturers. However, some things are handled differently, for example synchronized signals.

2. Wire types and constants
   
   In VHDL/Verilog every port/signal must have a type. Usually you would assign to a single wire connection a logical type that also handles a wide class of states other than `0` and `1`, for example: high impedance `Z`, uninitialized `U`, unknown/runtime error `X`, anything goes `-`. In VHDL you can use `std_logic`/`std_ulogic` type whereas in Verilag you can use `wire`. In this language such type is called `logic` like in Verilog.

   Currently all the following constants have type `logic`:

   + <code>`0</code>: logic state 0;
   + <code>`1</code>: logic state 1;
   + <code>`Z</code>: high impedance;
   + <code>`U</code>: uninitialized data;
   + <code>`X</code>: unknown value, runtime error;
   + <code>`-</code>: unknown value that can be safetly ignored.
     
   ```
       port1 = `1;    //assigns logic state 1 to port1
       port2 = `Z;     //assign high impedance state to port2
   ```

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

   You can access each element by specifying its index

       porta = ports[1];    //set porta std_ulogic signal to the element of ports with index 1

   or you can access a continuous range of elements with the : operator

       portt = ports[6:4];    //equivalent to portt[0]=ports[4]; portt[1]=ports[5]; portt[2]=ports[6];

   Ranges have the following format:

   *max_index* : *min_index*

   with *max_index* greater or equal to *min_index*. Moreover, you can access an array in a reverse order by using *inverse ranges* which can be defined in the following way:

   *min_index* :: *max_index*

   example:

       portt = portr[0::2];    //portt[0] = portr[2]; portt[1] = portr[1]; portt[2] = portr[0];

   Array values can be declared with `[`, `]` brackets with items separated by commas, however despite programming languages like C array indexing goes from right to left instead of from left to right:

       porta = [`0, `1, `Z];    //porta[2] = `0; porta[1] = `1; porta[0] = `Z;

   Moreover, logic array constants can be also declared between `"` symbols, again from right to left: 

       porttt = "01U";    //equivalent to portt[2]=`0; portt[1]=`1; portt[0]=`U;

   Logical integer constants (which are different from integer constants in array indices) can be assigned to logic arrays. They have the following format:

   *num_bits* *radix* *number*

   where *radix* can be `d` (decimal), `h` (hexadecimal) or `o` (octal). Less significant bits are assigned to lower indices first:

       porta = 3d3;    //porta[0] = `1; porta[1] = `1; porta[2] = `0;

4. Components and implementations

   A component in this language can be define in the following way: 
   <pre><code>
   comp <i>name</i> {
     in {
       //input ports
     }
     out {
       //output ports
     }
     trigger {
       //trigger ports
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
     trigger {
       //triggers
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

   Ports must be defined in `in` (for input ports), `out` (for output ports) or `trigger` (clock and reset signals) subblocks in `comp` or `design` blocks (*TODO*: introduce `inout` ports). A port definition follows this syntax:

   <pre><code>
      <i>port_name</i> : <i>type_name</i>;
   </code></pre>

   If you don't specify *type_name* then `logic` is assumed. Example:

       design multiplex_4_1 {
         in {
           adr : [2]logic;
           input : [4];
         }
         out {
           output;    //assumed logic
         }
       }
   
5. Signals and parallel coding

   Inside the `impl` subblock you put all the logical ports, connections and components you need in order to implement the behaviour of your component. This block consists in a list of assigments on the following form:

     *signal/out port name* = *signal expression*;

   All these statements represent wire connections so ordering doesn't matter, moreover these statements are continuously evaluated until each signal/output port has a value. For example with the following code

       A = B;
       B = `1;
       C = A;

   `A`, `B` and `C` would all be equal to <code>`1</code>.


   In VHDL/Verilog you must first define all your signals and declare their type before using it. Here signals aren't declared in a separate line, but they are defined when you use it for the first time. However, each signal/ouput port can appear at the left side of `=` symbol **only once**, so the following code would issue an error

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

     these operators works both on single `logic` types, arrays of `logic`, array of array of `logic` and so on, by applying the chosen operation on each element. However, both the arguments must have the same type because this language is strongly typed.
   - Range operator `[min:max]` as described before;
   - Array composition `[...]`:

         A = `0;  //type logic
         B = `1;  //ditto
         C = [A, B]; //equal to "01", type [2]logic
         D = [A];  //type [1]logic, which is different from logic
     
   - Array concatenation `~`:

         A = "0";  //type [1]logic
         B = "1";  //type [1]logic
         C = A ~ B;  //equal to "01", type [2]logic
  
   - Component allocation `new`. This operator is a bit complex and needs first an example. Suppose we have already defined the following adder component:

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
           trigger {
             clock;
           }
           impl {
             ...
           }
         }

     and we want to instantiate it in another component. You can do it thanks to the `new` operator in the following way:

         adr = new ADDER {
             a = C;
             b = D;
             in_rem = `0;
             clock = clock;
           };

     where `C`, `D`, `E` are other signals with compatible types. Now `adr` is not a `logic` or array signals, so you can put it directly in other signal expression. To take its output you must use the `.` operator:

         sig = adr.sum;    //reads the sum port of ADDER
         sig_rep = not adr.out_rem;  //uses the out_rem port of ADDER

     if you need only one output value then wyou can apply the `.` operator directly on the valued returned by `new`:

         sig = new ADDER {
             a = C;
             b = D;
             in_rem = `0;
             clock = clock;
           }.sum;

     The `new` operator also allows you to instantiate an array of components at the same time. Consider for example the following component:
  
         comp MPX {
           in {
             adr;
             data : [2];
           }
           out {
             bus;
           }
           trigger {
             clock;
           }
           impl { ... }
         }
  
     We can allocate three `MPX` components in this way:
  
         multiplex = new [3]MPX {adr = A; data = D; clock = clock;};
  
     which is equivalent to the following code
  
         multiplex = [ new MPX {adr = A[0]; data = D[0]; clock = clock;},
                        new MPX {adr = A[1]; data = D[1]; clock = clock;},
                        new MPX {adr = A[2]; data = D[2];}];
  
     Notice that when allocating `N` copies of `MPX` with a single `new` instruction the input port `adr` type becomes `[N]logic` and `data` type becomes `[N][2]logic` (a `[N]` is concatenated at each input type) but the trigger port `clock` has its type unchanged since the signal is shared between all the instances of `MPX`. Indeed the main difference between an `in` port and a `trigger` port is that `trigger` ports would be shared among all their instances after an array allocation, whereas `in` (and `out`) ports remain independent. 

     Now `multiplex` is an "array of components" each of them can be accessed by specifying its index:
  
         dat = multiplex[1].bus;
  
     Moreover, you can threat the entire signal `multiplex` (or even a range of it) as a single component with output ports type modified as the input ones. For example the following expression
  
         multiplex.bus
  
     is equivalent to
  
         [multiplex[0].bus, multiplex[1].bus, multiplex[2].bus]
     
7. `sync` signals and sequential coding

   Parallel coding defined in the previous part has a very restrictive requirement: no cyclic assigments. For example this code wouldn't compile

       A = B;
       B = A;

   because the value of `A` depends on `B` and vice versa. Cyclic connections are fundamental for sequential hardware, therefore VHDL/Verilog introduced `process` blocks that evaluated signal sequentially. This language instead uses a completely different approach by defining `sync` signals. A `sync` signal is defined in the following way:
   <pre><code>sync(<i>clock expression</i>) <i>signal_name</i> = <i>signal expression</i>;</code></pre>
   here *clock expression* is a regular signal expression that emits "events" which change the "state" of *signal_name*. For example let `clock` be a input port/signal, then `sync(clock) A = ...` not only defines the signal `A`, but also defines the "virtual input port" `^A` that holds the values of `A` *in the previous state* determined by `clock`. When `clock` emits an event (for example, changing from `0` to `1` or emits an impulse) then a state change occurs, in which the current value of `A` is moved into `^A` and then a new value of `A` is computed. You can use `A` and `^A` in almost (*TODO*: analyse all scenarios) every signal expression in your component, but you *can't* manually assign a new value to `^A` because it is an input port and not a signal.

   The following code compiles correctly:

       sync(clock) A = B;
       B = ^A;

   because now `A` and `^A` are different. Since the assignations `A = B` and `B = ^A` are immediate then `A` would always be equal to both `B` and `^A`. If you want instead to exchange `A` with `B` then a simple but unstable way to do it is

       sync(clock) A = ^B;
       sync(clock) B = ^A;

   However, this could be not suitable for real hardware because hardware delays in `clock` signal may pollute both `A` and `B`.
   
9. Automatic code generation

    The preceding instructions allows you to design your hardware. However, for real projects you have to deal with tons of signals and ports or repetitive code. For these reasons a second-order language has been developed in order to help programmers to generate large values of repetitive code.

   In this new language instructions are evaluated sequentially, after the code has been scanned and tokenized but before wire code is interpreted. Variables here must start with the `$` symbol and currently they can be only boolean, nonnegative integers and range specifications:

       $var = true;    //boolean variable
       $num = 3;    //integer variable.
       $range = 3:0    //range

   You can use standard arithmetic operations `+, -, *, /`, logic operations `&&, ||, !`, comparisons `<, >, ==, !=, <=, >=`, the length function `#len(...)` that returns the length of specified range and the *range shift* operations `>>, <<` that shift the specified ranges:

       2:0 << 3    //expands to 5:3
       7:3 >> 1    //expands to 6:2
       3::5 << 2   //expands to 1::3

   *TODO*: introduce functions. The `=` operator assign values to variables. You can place these variables in wire code in order to place their value inside the code. Let for example

       port[$x]

   where `$x` is a variable previously assigned. Indeed since variables are evaluated sequentially ordering here matters, so in order to use `$x` in your wire code you sould have assigned a value at least once. For example if we've set `$x = 2;` previously then the preceding code would become

       port[3]

   during wire code interpretation. But if we have set `$x = 4:0;` then it becomes

       port[4:0]

   which is still valid. To conditionally evaluate code you can use the following *if* statement:
   <pre><code>
      #if <i>expression</i> {
      ...
      }
      #elif <i>expression</i> {
      ....
      }
      ....
      #else {
      ....
      }
   </code></pre>
   like in many other programming languages.

   Use of `[..]` and `~` to assign array values can be not so practical in certain situations. In such cases you can use the `#array` function

       portt = #array {
         0 = `1;
         4::5 = [idx, `0];
         1 = tris[1] or tris[0];
         3:2 = "10";
       };

       //which is equivalent to

       portt = [`0, idx] ~ "10" ~ [tris[1] or tris[0], `1];

   To repeat code you can use the `#repeat` statement:
   <pre><code>
      #repeat $index in <i>range expression</i> {
      ....
      }
   </code></pre>
   that works like a classical `for` loop and concatenates each value. Moreover, you can also use the `#repeat_range` expression:
   <pre><code>
      #repeat_range $range in (<i>range expr</i>; <i>len expr</i>) {
      ....
      }
   </code></pre>
   which divides *range expr* into many subranges of length __at most__ *len expr* and assigns each to `$range` sequentially. For example

       #repeat_range $rng in ([12:0]; 5) {
         port[$rng] xor tris[$rng << 1]
       }
   
   expands to

       (port[12:10] xor tris[13:11])
       ~ (port[9:5] xor tris[10:6])
       ~ (port[4:0] xor tris[5:1])

11. Template parameters and autodetection

    *Work in progress*.
