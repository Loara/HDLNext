## Why a new HDL language?
   
   There isn't a specific reason. In fact, if you're working for a company you don't usually have the right to choose an HDL language to develop hardware, therefore you would necessarly use VHDL or (System)Verilog. Even if you're an hobbist you have to use a well-known language because almost every library you need is written in one of the previous languages.

   However, VHDL and Verylog have been developed several years ago, so their syntax in 2024 are quite outdated. But instead of define a new HDL language that will never supersede VHDL/Verilog it is more reasonable to introduce a "VHDL/Verilog language generator" that allows you to design hardware with a more modern and readable syntax and then use it to generate VHDL/Verilog code that you can feed to your favorite toolchain.

   **EDIT:** I've taken a look at Lucid, which have a very similar syntax to this language and tries to achieve the same goal but it has already been adopted by some FPGA manifacturers. However, some things are handled differently, for example synchronized signals.

[Next](types.md)
