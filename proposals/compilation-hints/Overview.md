# Compilation Hints Proposal

## Summary

This proposal allows WebAssembly modules to provide optional hints that help the compiler take the best decisions for the intended use case. These are mere performance improvements and there is no change in functionality or behavior.


## Motivation

Ensuring scalability for WebAssembly applications often requires engines to take decisions without having all the information available, e.g. when to compile a function in which tier. These are usually driven by heuristics and feedback collected at runtime. Heuristics are only a broad approximation of the real-world behavior and runtime feedback requires time and potentially expensive instrumentation to gather. It can therefore beneficial to provide additional information of the expected usage pattern to the compiler beforehand.

This information can be extracted from source code, AOT analysis or profiling. AOT optimizers can potentially also use this information to take advanced decisions where runtime feedback would normally not be available. They can then decide to act on that information or forward it to the runtime engine.

One interesting aspect to other environments where profiling feedback is to be incorporated is that this usually requires recompiling the code from source and therefore access to the source code while WebAssembly allows to add information after module creation and then re-optimize modules or rely on the engine to incorporate the feedback. This can be useful for libraries that are used in different scenarios where different compilation patterns are preferred.


## Overview

Based on the [branch hinting proposal](https://github.com/WebAssembly/branch-hinting), we extend this mechanism by adding additional custom sections for extra functionality. This can be integrated with the [annotations proposal](https://github.com/WebAssembly/annotations) for the ability to generate these from text format and for round-trips to the text format to preserve them.

Each family of hints is bundled in a respective custom section following the example of branch hints. These sections all have the naming convention `metadata.code.*` and follow the structure
  * *function index* |U32|
  * a vector of hints with entries
    * *byte offset* |U32| of the hinted instruction from the beginning of the function body (0 for function level hints) (the function body begins at the first byte of its local declarations, the same as branch hints),
    * *hint length* |U32| indicating the number of bytes each hint requires,
    * *values* |U32| with the actual hint information

If not specified otherwise, all numeric values are encoded using the [LEB128](https://en.wikipedia.org/wiki/LEB128) variable-length integer encoding either in its signed or unsigned variant.

*Note: If custom annotations support a `metadata.function.*` namespace, the byte offset could be dropped for function-level annotations.*

The following contains a list of hints to be included in the first version of the proposal. Future extensions can be added as necessity arises. This also includes annotations outside of function or instruction level like annotations for memories, etc.


### Compilation order

The section `metadata.code.compilation_order` contains the order in which functions should be compiled in order to minimize wait times until the compilation is completed. This is especially relevant during instantiation and startup but might also be relevant later.
  * *byte offset* |U32| with value 0 (function level only)
  * *hint length* |U32| in bytes
  * *compilation priority* |U32| starting at 0 (functions with the same order value will be compiled in an arbitrary order but before functions with a higher order value)
  * *hotness* |U32| defining how often this function is called

If a length of larger than required to store 2 values is present, only the first two values of the following hint data is evalued while the rest is ignored. This leaves space for future extensions, e.g. grouping functions. Similarly, the *hotness* can be dropped if a length corresponds to only 1 value is given.

The *hotness* attribute has no pre-defined meaning. The larger the value, to more often a function is expected to run. So an engine can simply order the functions by hotness and tier up the ones with the largest *hotness* until the compilation budget is exceeded. The compilation budget might depend on the engine, compiler, available resources, how long the program has been running, etc. The special value of 0 is reserved for functions that only run once (e.g. initialization functions). An engine can decide to interpret those functions only or to free up code space by removing the compiled code after execution. Applications can run such a function multiple times, but they should not because this might come with severe performance penalties, e.g. for repeated recompilation, not ever getting tiered up, etc.

It is expected and even desired that not all functions are annotated to keep this section small. It is up to the engine if and when the unannotated functions are compiled. It's recommended that these functions get compiled last or lazily on demand.

*Note: This should be moved to `metadata.function.compilation_order` without the byte offset if such a namespace will be supported by custom annotations.*


#### Text format

Instead of a binary string representation, these hints can also be provided using a more human readable notation in text format:
```
(@metadata.code.compilation_order (priority 1) (hotness 100))
```
The above example is equivalent to
```
(@metadata.code.compilation_order "\01\64")
```
and tools can produce one or the other.


### Instruction frequencies

Instruction frequencies might be useful to guide optimizations like inlining, loop unrolling, block deferrals, etc. Within a function, these frequencies inform which blocks lie on the hot path and deserve more expensive optimizations, as well as which are on the cold path and might even allow very expensive steps to even execute the code within (e.g. outlining or de-optimization). An engine can take those decisisions based on the instruction frequency observed, but cannot assume that any part of the code is unreachable based on the instruction frequency.

The `metadata.code.instr_freq` section contains instruction level annotations for optimizable instruction sequences.
  * *byte offset* |U32| from the beginning of the function to the wire byte index of an instruction (e.g. start of a loop or a block containing a call instruction).
  * *hint length* |U32| in bytes (always 1 for now, might be higher for future extensions)
  * *offset log2 frequency* |U8| determining the estimated number of times the instruction gets executed per execution of the containing function.

Instruction frequencies are always relative to the surrounding function and therefore every instruction at the beginning of a function has an implicit frequency of 1 assigned to it. Each annotation remains valid for all following instructions until the next control flow instruction (`br`, `br_table`, `br_if`, `if`, `return`, `unreachable`, `br_catch`, `throw`).

For now, we expect annotations to only affect `call`, `call_ref`, `call_indirect` and `loop` instructions. Tools can focus on providing annotations for these instruction to make sure they have the expected impact without unnecessary binary bloat. In the future, this can be extended to more instructions if needed. The structure and the name of this annotation has been specifically chosen to allow for that.

The instruction frequency can be thought of the estimated number of times an instruction gets executed during one call of the function. It is a logarithmic value based on the formula $f = \max(1, \min(64, \lfloor \log_{2} \frac{n}{N} \rfloor + 32))$ where $n$ is the total number of executions of this instruction and $N$ is the number of calls to this function.

The main expected use of this hint by engines it to guide inlining decisions. However the actual decision which function should be inlined can be based on runtime data that the engine collected, additional heuristics and available resources. There is no guarantee that a function is or is not inlined, but it should roughly be expected that functions of higher call frequency are prefered over ones with lower frequency.
Special values of 0 and 127 indicate that a function should never or always be inlined respectively. Engines should respect such annotations over their own heuristics and toolchains should therefore avoid generating such annotations unless there is a good reason for it (e.g. "no inline" annotations in the source).

|binary value|log2 frequency|executions per call|
|-----------:|-------------:|:------------------:|
|           0|              |*never optimize*    |
|           1|           -31|          <9.313e-10|
|           8|           -24|           5.960e-08|
|          16|           -16|           1.526e-05|
|          24|            -8|           0.00391  |
|          28|            -4|           0.0625   |
|          30|            -2|           0.25     |
|          31|            -1|           0.5      |
|          32|             0|           1        |
|          33|            +1|           2        |
|          34|            +2|           4        |
|          36|            +4|          16        |
|          40|            +8|         256        |
|          48|           +16|       65536        |
|          56|           +24|           1.678e+07|
|          64|           +32|          >4.295e+09|
|         127|              |*always optimize*   |


#### Text format

The alternative text format representation for such a section would look as follows
```
(@metadata.code.instr_freq (freq 123.45))
```
The given frequency $\frac{n}{N}$ is then converted into the equivalent binary representation
```
(@metadata.code.instr_freq "\26")
```
according to the formula above.

Alternatively to `freq` followed by a numeric value, one can provide `never_opt` and `always_opt` with binary representations of `"\00"` and `"\7f"` respectively.


### Call targets

When dealing with `call_indirect` or `call_ref`, often inefficient code is generated, because the call target is unknown. With code that e.g. uses virtual function calls, there are often very few commonly called targets which a compiler could optimize for. It still needs to have the ability to handle other call targets, but that can then happen at a much lower performance in favor of optimizing for the more commonly called target.

This is especially interesting if functions need to be compiled to the top tier early on, either because they're annotated with a low compilation order, because eager compilation or even AOT compilation is desired.

The `metadata.code.call_targets` section contains instruction level annotations for all relevant call targets identified by their function indexes.
  * *byte offset* |U32| from the beginning of the function to the wire byte index of the call instruction (this must be a `call_ref` or a `call_indirect`, otherwise the hint will be ignored)
  * *hint length* |U32| in bytes
  * call target information
    * *function index* |U32|
    * *call frequency* |U32| in percent

The accumulated call frequency must add up to 100 or less. If it is less than 100, then other call targets that are not listed are responsible for the missing calls. Together with the inline hints on call frequency, this can information can be used to inline function calls as well. The effective call frequency for each call target is then the inlining call frequency multiplied by the fractional call frequency encoded in this section.

Similarly to the compilation order section, not all call sites need to be annotated and not all call targets be listed. However, if other call targets are known but not emitted, then the frequency must be below 100 to inform the engine of the missing information.


#### Text format

The text representation allows for multiple targets
```
(@metadata.code.call_targets (target $func1 0.73) (target $func2 0.21))
```
which would be converted into binary format as
```
(@metadata.code.call_targets "\01\49\02\15)
```
under the assumption that `$func1` and `$func2` have function indices 1 and 2 respectively.
