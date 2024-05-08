## Expressions

### logic constants

Currently all the following constants have type `logic`:

+ <code>`0</code>: logic state 0;
+ <code>`1</code>: logic state 1;
+ <code>`X</code>: unknown value, runtime error;

Example:
<pre><code>
port1 = `1;    //assigns logic state 1 to port1
port2 = `X;     //assign runtime error to port2 which propagates to other components
</code></pre>

In order to combine multiple `logic` literals into a single array literal of type `[N]logic` you can enclose the ordered logic literals with `"` in the following way:

    port = "01X";    //port has type [3]logic with port[0] = `X, port[1] = `1 and port[2] = `0

### array expressions

You can access each element by specifying its index

    porta = ports[1];    //set porta std_ulogic signal to the element of ports with index 1

or you can access a continuous range of elements with the : operator

    portt = ports[6:4];    //equivalent to portt[0]=ports[4]; portt[1]=ports[5]; portt[2]=ports[6];

Ranges have the following format:

<pre><code>
    <i>max_index</i> : <i>min_index</i>
</code></pre>

with *max_index* greater or equal to *min_index*. Moreover, you can access an array in a reverse order by using *inverse ranges* which can be defined in the following way:

<pre><code>
    <i>min_index</i> :: <i>max_index</i>
</code></pre>

example:

    portt = portr[0::2];    //portt[0] = portr[2]; portt[1] = portr[1]; portt[2] = portr[0];

Array values can be declared with `[`, `]` brackets with items separated by commas, however despite programming languages like C array indexing goes from right to left instead of from left to right:

    porta = [`0, `1, `Z];    //porta[2] = `0; porta[1] = `1; porta[0] = `Z;

## fields
You can access a field with the `.` operator:

    A.fld;  //access field fld of A

### logical ports
You can use logical ports `$and`, `$or`, `$nand`, `$nor`, `$xor`, `$xnor` as follows:

    A = $and(B.not, C, $xor(D, `1));

these operators works both on single `logic` types, arrays of `logic`, array of array of `logic` and so on, by applying the chosen operation on each element. However, both the arguments must have the same type because this language is strongly typed.

### Array concatenation `$cat`:

    A = "0";  //type [1]logic
    B = "1";  //type [1]logic
    C = $cat(A , B);  //equal to "01", type [2]logic
  
### Component allocation `new`

To instantiate a prodtype/sumtype/component in an expression you can use the `new` operation. For example assume we have defined the following prodtype:

    prodtype sum {
        s,
        c_out,
    }

then you can allocate it inside a component in this way:

    a = new sum {
        s = A,                  //connect A to s
        c_out = $or(B, C.not),  //connect $or(B, C.not) to c_out
    };
    b = a.s;                    //access field s in a which has type sum

For sumtypes you need to specify exactly one of its item in the `new` block. If the chosen item has type `void` then you can avoid using the `{..}` block:

    sumtype B {
        itemA : [2],
        itemB,          //void
    }
    ..
    C = new B {
        itemA = "10"
    };
    D = new B itemB;

The `new` operator can also instantiate components into other components, in that case you need to connect input ports during allocation. For example consider the following full adder component definition: 

    comp FULL_ADDER {in a, in b, in c_in} -> sum_res {
        d = a xor b; 
        . = new sum {
            s = d xor c_in,
            c_out = (a and b) or (d and c_in),
        };
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

The `new` operator also allows you to instantiate an array of prodtypes/sumtypes/components at the same time. Consider for example the following component:
  
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

[Prev](comp.md) [Next](sync.md)
