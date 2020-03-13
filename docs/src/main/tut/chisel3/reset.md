---
layout: docs
title:  "Reset"
section: "chisel3"
---
As of Chisel 3.2.0, Chisel 3 supports both synchronous and asynchronous reset,
meaning that it can natively emit both synchronous and asynchronously reset registers.

The type of register that is emitted is based on the type of reset associated with that register by the usual Chisel rules.

Signals used as resets now have a trait `Reset`.
There are three types which implement the trait `Reset`:
`Bool`, `AsyncReset`, and `ResetType`.
Registers which are ultimately driven by a `Bool` are emitted as synchronous reset flops,
and registers which are driven by a `AsyncReset` are emitted as asynchronouly reset flops.
Registers which are driven by a `ResetType` don't make sense and will cause an error if attempted.


A `Module`'s `reset` is of abstract type `Reset`.
Previous to Chisel 3.2.0 a `Module`'s `reset` was of type `Bool`,
so for backwards compatability the top-level module in a Chisel design's `reset` is driven by type `Bool`.

Consider the two example modules below which are agnostic to the type of reset used within them:

```
class ResetAgnosticModule extends Module() {
  val io = IO(
    val out = UInt(4.4) 
  )
  val reset_agnostic_reg = RegInit(0.U(4.W))
  reset_agnostic_reg := reset_agnostic_reg + 1.U
  io.out := reset_agnostic_reg
}

class ResetAgnosticRawModule extends RawModule {
  val clk = IO(Input(Clock()))
  val rst = IO(Input(Reset()))
  val out = IO(Output(UInt(8.W)))

  val reset_agnostic_reg = withClockAndReset(clk, rst)(RegInit(0.U(8.W)))
  reg := reg + 1.U
  out := reset_agnostic_reg
}
```
To make registers get the desired kind of reset,
you can convert into the correct reset types as follows.

This will make both `reset_agnostic_reg` to be synchronously reset:

```
withReset(reset.asBool){
  val myModule = Module(new ResetAgnosticModule)
  val myRawModule = Module(new ResetAgnosticRawModule)
  myRawModule.rst := reset
  myRawModule.clk := clock
}  
```

This will make both `reset_agnostic_reg` to be asynchronously reset:

```
withReset(reset.asAsyncReset){
  val myModule = Module(new ResetAgnosticModule)
  val myRawModule = Module(new ResetAgnosticRawModule)
  myRawModule.rst := reset
  myRawModule.clk := clock
}
```

If you would like to enforce the reset type from within a Module,
vs from its instantiator, you can use the `Requires*Reset` traits as follows:

```
class MyAlwaysSyncResetModule extends Module with RequireSyncReset {
  val my_sync_reset_reg = RegInit(false.B)
}

class MyAlwaysAsyncResetModule extends Module with RequireAsyncReset {
  val my_async_reset_reg = RegInit(false.B)
}
```
