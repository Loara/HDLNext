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
