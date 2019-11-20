---
layout: docs
title:  "Reset"
section: "chisel3"
---
As of Chisel 3.2.0, Chisel 3 supports both synchronous and asynchronous reset.
It can emit both synchronous and asynchronously reset registers.

The type of register that is emitted is based on the type of reset associated with it by the usual Chisel rules.

Signals used as resets now have three types:
`Bool` (Synchronous Reset), or `AsyncReset` (AsynchronousReset), or `Reset` (inferred from driving signal). 

A `Module`'s default `reset` is of type `Reset`.
(Note that previous to Chisel 3.2.0 a `Module`'s default `reset` was of type `Bool`).

At elaboration time, the Module's reset must be of a known type of `Bool` or `AsyncReset` so that the flops can be emitted.

To make registers get the right kind of reset, you can convert into the correct reset types as follows:

```
class MyModule extends Module() {
  val io = IO(
    val out = UInt(4.4) 
  )
  val dont_care_reset_reg = RegInit(0.U(4.W))
  dont_care_reset_reg := dont_care_reset_reg + 1.U
  io.out := dont_care_reset_reg
}
```

This will make `dont_care_reset_reg` to be synchronously reset:

```
val syncReset = reset.toBool

val myModule = Module(new MyModule)
```

This will make `dont_care_reset_reg` to be asynchronously reset:

```
val syncReset = reset.asAsyncReset

val myModule = Module(new MyModule)
```

If you try to make a register when the reset is just `Reset` you will get an error.
