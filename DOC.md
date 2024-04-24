1. Why a new HDL language?
   
   There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

   However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

   **EDIT:** I've taken a look at Lucid, which have a very similar syntax to this language and tries to achieve the same goal but it has already been adopted by some FPGA manifacturers. However, some things are handled differently, for example synchronized signals.

2. Wire types and constants

   This language is strongly types, that is every port and every wire must have a type and two wires/ports can be connected if and only if they have the same type. Currently the following types are defined:

   + Basic types
     - `logic`
     - `void`
   + Arrays
   + Struct
   
   In VHDL/Verilog every port/signal must have a type. Usually you would assign to a single wire connection a logical type that also handles a wide class of states other than `0` and `1`, for example: high impedance `Z`, uninitialized `U`, unknown/runtime error `X`, anything goes `-`. In VHDL you can use `std_logic`/`std_ulogic` type whereas in Verilag you can use `wire`. In this language such type is called `logic` like in Verilog.

   Currently all the following constants have type `logic`:

   + <code>`0</code>: logic state 0;
   + <code>`1</code>: logic state 1;
   + <code>`X</code>: unknown value, runtime error;
     
   ```
       port1 = `1;    //assigns logic state 1 to port1
       port2 = `X;     //assign runtime error to port2 which propagates to other components
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

       porttt = "01X";    //equivalent to portt[2]=`0; portt[1]=`1; portt[0]=`X;

   Logical integer constants (which are different from integer constants in array indices) can be assigned to logic arrays. They have the following format:

   *num_bits* *radix* *number*

   where *radix* can be `d` (decimal), `h` (hexadecimal) or `o` (octal). Less significant bits are assigned to lower indices first:

       porta = 3d3;    //porta[0] = `1; porta[1] = `1; porta[2] = `0;

   The last basic type is `void`. This type is very strange because it has only one possible value, which is `""`. Since it doen't hold or bring any information signals and ports of such type are never synthetized and can be safetly ignored when the component is instantiated. However, this type is useful in metaprogramming due to the following conversion rules:
   + `[N] void` is equal to `void` for every `N >= 0`;
   + `[0] ty` is equal to `void` for any type `ty`;
   + if `M < N` then `port[M:N]` and `port[N::M]` are equal to `""`.

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

4. Components

   A component in this language can be define in the following way: 
   <pre><code>
   comp <i>name</i> { <i>input ports</i> } -> <i>return type</i> {
     <i>assigments</i>
   }
   </code></pre>
   which is very different from VHDL/Verilog. Each assigment in component body must be terminated by `;` and can assume one of the following forms:

   <pre><code>
     <i>signal name</i> = <i>expression</i>  //simple signal assigment
     sync(<i>clock</i>) <i>signal name</i> = <i>expression</i>  //synchronized signal assigment
     . = <i>expression</i>  //output assignemt
   </code></pre>

   Only the output assigment must be present exactly once in component body, and its expression must have <i>return type</i> as type. However, if <i>return type</i> is a struct with fields <i>fd1</i>, <i>fd2</i>, <i>fd3</i>, ..., <i>fdn</i> then the output assigment can be replaced by all these assigments:
   <pre><code>
   .fd1 = <i>expr1</i>
   .fd2 = <i>expr2</i>
   .fd3 = <i>expr3</i>
   //...
   .fdn = <i>exprn</i>
   </code></pre>

   An input port definition must be one of the followings:

   <pre><code>
      in <i>port_name</i> : <i>type_name</i>
      shr <i>port_name</i> : <i>type_name</i>
   </code></pre>

   and are separated by commas. If you don't specify *type_name* then `logic` is assumed. Example:

       design multiplex_4_1 {in adr : [2]logic, in input : [4], shr clock}

   The difference between an `in` port and a `shr` port appears when a component is instantiated with `new`. Otherwise they are equivalent.

6. Signals and parallel coding

   Simple signal and output assigments are "executed" at the same time, so ordering doesn't matter here. For example

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
  
   - Component allocation `new`. This operator is a bit complex and needs first an example. Suppose we have the following full adder:

         struct sum {
           s,
           c_out,
         }

         comp ADDER {in a, in b, in c_in} -> sum_res {
           d = a xor b; 
           .s = d xor c_in;
           .c_out = (a and b) or (d and c_in);
         }

     and we want to instantiate it inside another component. You can do it thanks to the `new` operator in the following way:

         adr = new ADDER {
             a = C;
             b = D;
             c_in = `0;
           };

     where `C`, `D`, `E` are other signals with compatible types. Now `adr` has type `sum` so you can access its fields with the `.` operator:

         sig = adr.s;    //reads the s port of ADDER
         sig_rep = not adr.c_out;  //uses the c_out port of ADDER

     you can apply the `.` operator directly on the valued returned by `new`:

         sig = new ADDER {
             a = C;
             b = D;
             c_in = `0;
           }.s;

     The `new` operator also allows you to instantiate an array of components at the same time. Consider for example the following component:
  
         comp MPX {shr adr, in data : [2]} -> logic {
           . = (adr and data[1]) or (adr.not and data[0]);
         }
  
     We can allocate three `MPX` components in this way:
  
         multiplex = new [3]MPX {adr = A; data = D;};
  
     which is equivalent to the following code
  
         multiplex = [ new MPX {adr = A; data = D[2]; },
                        new MPX {adr = A; data = D[1]; },
                        new MPX {adr = A; data = D[0]; }];
  
     Notice that when allocating `N` copies of `MPX` with a single `new` instruction the `in` port `data` type becomes `[N][2]logic` (an `[N]` is concatenated at each in type) but the `shr` port `adr` has its type unchanged since the signal is shared between all the instances of `MPX`. Indeed the main difference between an `in` port and a `shr` port is that `shr` ports would be shared among all their instances after an array allocation, whereas `in` ports remain independent. `shr` ports are well-suited for clock signals. 

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
   
8. Metalanguage

    The preceding instructions allows you to describe your components and interconnections. However, for real projects you have to deal with tons of signals and ports or repetitive code. For these reasons a second-order language has been developed in order to help programmers to generate large values of repetitive code. This language is called *metalanguage* for simplicity

   In metalanguage instructions are evaluated sequentially, after the code has been scanned and tokenized but before wire code is interpreted. Variables are declared with the following syntax:

       #let var = expression;

   and currently they can be only
   + booleans
   + nonnegative integers
   + ranges
   + components
  
  For example:   

       #let var = true;    //boolean variable
       #let num = 3;    //integer variable.
       #let range = 3:0;    //range
       #let comp = MULTIPLEX;    //a component

   You can use standard arithmetic operations `+, -, *, /`, logic operations `&&, ||, !`, comparisons `<, >, ==, !=, <=, >=`, the length function `$len(...)` that returns the length of specified range and the *range shift* operations `>>, <<` that shift the specified ranges:

       2:0 << 3    //expands to 5:3
       7:3 >> 1    //expands to 6:2
       3::5 << 2   //expands to 1::3

   *TODO*: introduce functions. You can evaluate a metalanguage expression and put its value in your code by enclosing the expression between `$(` annd `)`:

       $( expression )

   when `expression` is just a single variable then you can remove `(` and `)`. Let for example

       port[$x]

   and assume we have set `#let x = 3;` before. Then after metalanguage evaluation the preceding line becomes

       port[3]

   But if we have set `#let x = 4:0;` then it becomes

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
      #repeat <i>index</i> in <i>range expression</i> {
      ....
      }
   </code></pre>
   that works like a classical `for` loop and concatenates each value with `~`. For example

       #repeat i in [4::0] {
         [aut[$i]]
       }
       
   is equivalent to

       [aut[4]] ~ [aut[3]] ~ [aut[2]] ~ [aut[1]] ~ [aut[0]]
       

9. Template parameters and autodetection

    Components can have template parameters, which are metalanguage variables declared in a `comp` block between `!(` and `)` after the component name. The following code define Multiplex components for an arbitrary number of inputs:

        design multiplex_2_1 {
          in {
            adr;
            in0;
            in1;
          }
          out {
            out;
          }
        }
    
        comp MULTIPLEX ! (
          N : integer;    //N is an integer variable
          M : design multiplex_2_1;  //M is a component variable
        ){
          in {
            adr : [$N];
            val : [$(pow2(N))];
          }
          out {
            out;
          }
          impl {
            #if N == 0 {
              out = val[0];
            }
            #else {
              T = new MULTIPLEX!(N-1){
                adr = adr[$(N-2):0];
                val = val[$(pow2(N-1)-1):0];
              };
              B = new MULTIPLEX!(N-1){
                adr = adr[$(N-2):0];
                val = val[$(pow2(N)-1):$(pow2(N-1))];
              };
              out = new $M {
                adr = adr[$(N-1)];
                in1 = T.out;
                in0 = B.out;
               }.out;
            }
          }
        }
