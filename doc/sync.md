## `sync` signals and sequential coding

Combinatorial assigments have a very restrictive requirement: no cyclic assigments. For example this code wouldn't compile

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
 
[Prev](expr.md)
