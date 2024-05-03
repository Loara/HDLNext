## Expressions

### logic constants

   Currently all the following constants have type `logic`:

   + <code>`0</code>: logic state 0;
   + <code>`1</code>: logic state 1;
   + <code>`X</code>: unknown value, runtime error;
     
   ```
       port1 = `1;    //assigns logic state 1 to port1
       port2 = `X;     //assign runtime error to port2 which propagates to other components
   ```

   In order to combine multiple `logic` literals into a single array literal of type `[N]logic` you can enclose the ordered logic literals with `"` in the following way:

   ```
       port = "01X";    //port has type [3]logic with port[0] = `X, port[1] = `1 and port[2] = `0
   ```

### array expressions

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

## fields
You can access a field with the `.` operator:

```
A.fld;  //access field fld of A
```

Not only struct, but also every entity of type `logic` (or array of `logic`, array of arrays of `logic` and so on) have accessible fields. At the moment the only accessible field for them is `not` that returns its logic negation:

```
a = "1011X";
b = a.not;  //b has type [5]logic and it is equal to "0100X"
```

### logical ports
You can use logical ports `$and`, `$or`, `$nand`, `$nor`, `$xor`, `$xnor` as follows:

         ...
         A = $and(B.not, C, $xor(D, `1));
         ...

     these operators works both on single `logic` types, arrays of `logic`, array of array of `logic` and so on, by applying the chosen operation on each element. However, both the arguments must have the same type because this language is strongly typed.

### Array concatenation `$cat`:

         A = "0";  //type [1]logic
         B = "1";  //type [1]logic
         C = $cat(A , B);  //equal to "01", type [2]logic
  
### Struct/component allocation `new`

This operator is a bit complex and needs first an example. Suppose we have the following full adder:

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

     With `new` you can also allocate structs as they're components:

     ```
     struct Adr_2 {
        adr : [2],
     }
     ...
     ad = new Adr_2 {
        adr = ...;
     }
     x = ad.adr;
     ```

