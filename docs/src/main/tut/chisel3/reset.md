---
layout: docs
title:  "Reset"
section: "chisel3"
---
As of Chisel 3.2.0, Chisel 3 supports both synchronous and asynchronous reset,
meaning that it can natively emit both synchronous and asynchronously reset registers.

The type of register that is emitted is based on the type of the reset signal associated
with the register.

There are three types of reset that implement a common trait `Reset`:
* `Bool` - constructed with `Bool()`. Also known as "synchronous reset".
* `AsyncReset` - constructed with `AsyncReset()`
* `Reset`* - constructed with `Reset()`. Also known as "abstract reset".

\* For implementation reasons, the concrete Scala type is `ResetType`. Stylistically we avoid `ResetType`, instead using the common trait `Reset`.

Registers with reset signals of type `Bool` are emitted as synchronous reset flops.
Registers with reset signals of type `AsyncReset` are emitted as asynchronouly reset flops.
Registers with reset signals of type `Reset` will have their reset type _inferred_ during FIRRTL compilation.

### Reset Inference

FIRRTL will infer a concrete type for any signals of type abstract `Reset`.
The rules are as follows:
1. An abstract `Reset` with only signals of type `AsyncReset`, abstract `Reset`, and `DontCare`
in both its fan-in and fan-out will infer to be of type `AsyncReset`
2. An abstract `Reset` with signals of both typs `Bool` and `AsyncReset` in its fan-in and fan-out
is an error.
3. Otherwise, an abstract `Reset` will infer to type `Bool`.

You can think about (3) as the mirror of (1) replacing `AsyncReset` with `Bool` with the additional
rule that abstract `Reset`s with neither `AsyncReset` nor `Bool` in their fan-in and fan-out will
default to type `Bool`.
This "default" case is uncommon and implies that reset signal is ultimately driven by a `DontCare`.

### Module Implicit Reset

A `Module`'s `reset` is of type abstract `Reset`.
Prior to Chisel 3.2.0, a `Module`'s `reset` was of type `Bool`,
so for backwards compatability, the top-level reset will default to type `Bool`.

If you would like to set the reset type from within a Module (including the top-level `Module`),
rather than relying on Reset Inference, you can mixin one of the following traits:
* `RequireSyncReset` - sets the type of `reset` to `Bool`
* `RequireAsyncReset` - sets the type of `reset` to `AsyncReset`

For example:

```scala
class MyAlwaysSyncResetModule extends Module with RequireSyncReset {
  val mySyncResetReg = RegInit(false.B) // reset is of type Bool
}

class MyAlwaysAsyncResetModule extends Module with RequireAsyncReset {
  val myAsyncResetReg = RegInit(false.B) // reset is of type AsyncReset
}
```

### Examples

Consider the two example modules below which are agnostic to the type of reset used within them:

```scala
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

```scala
withReset(reset.asBool){
  val myModule = Module(new ResetAgnosticModule)
  val myRawModule = Module(new ResetAgnosticRawModule)
  myRawModule.rst := reset
  myRawModule.clk := clock
}  
```

This will make both `reset_agnostic_reg` to be asynchronously reset:

```scala
withReset(reset.asAsyncReset){
  val myModule = Module(new ResetAgnosticModule)
  val myRawModule = Module(new ResetAgnosticRawModule)
  myRawModule.rst := reset
  myRawModule.clk := clock
}
```

Note, it is *not* legal to override the reset type using last-connect semantics,
unless you are overriding a `DontCare`:

```scala
class MyModule extends Module {
  val reset_bool = Wire(Reset())
  reset_bool := DontCare 
  reset_bool := false.B // this is fine
  withReset(reset_bool) {
    val my_submodule = Module(new Submodule())
  }
  reset_bool = true.B // this is fine
  reset_bool = false.B.asAsyncReset // this is not fine
}
```
