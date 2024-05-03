## Components

   A component in this language can be define in the following way: 
   <pre><code>
   comp <i>name</i> { <i>input ports</i> } -> <i>return type</i> {
     <i>assigments</i>
   }
   </code></pre>
   which is very different from VHDL/Verilog. Each assigment in component body must be terminated by `;` and can assume one of the following forms:

   <pre><code>
     <i>signal name</i> = <i>expression</i>  //simple signal assigment
     sync(<i>clock</i>) <i>signal name</i> : <i>signal type</i> = <i>expression</i>  //synchronized signal assigment
     . = <i>expression</i>  //output assignemt
   </code></pre>

   Only the output assigment must be present in the component body and cannot be repeated multiple times. Moreover, its expression must have <i>return type</i> as type. 

    An input port definition must be one of the followings:

   <pre><code>
      in <i>port_name</i> : <i>type_name</i>
      shr <i>port_name</i> : <i>type_name</i>
   </code></pre>

   and are separated by commas. If you don't specify *type_name* then `logic` is assumed. Example:

       design multiplex_4_1 {in adr : [2]logic, in input : [4], shr clock}

   The difference between an `in` port and a `shr` port appears when a component is instantiated with `new` inside an expression. Otherwise they are equivalent.

[Prev](types.md) [Next](expr.md)
