# Debugging

Even if your kernel executes, it may be computing the wrong values, or even error at run
time. To debug these issues, both CUDA.jl and the CUDA toolkit provide several utilities.
These are generally low-level, since we generally cannot use the full extend of the Julia
programming language and its tools within GPU kernels.


## Adding output statements

The easiest, and often reasonably effective way to debug GPU code is to visualize
intermediary computations using output functions. CUDA.jl provides several macros that
facilitate this style of debugging:

- `@cushow` (like `@show`): to visualize an expression, its result, and return that value.
  This makes it easy to wrap expressions without disturbing their execution.
- `@cuprintln` (like `println`): to print text and values. This macro does support string
  interpolation, but the types it can print are restricted to C primitives.

The `@cuassert` macro (like `@assert`) can also be useful to find issues and abort execution.


## Stack trace information

If you run into run-time exceptions, stack trace information will by default be very
limited. For example, given the following out-of-bounds access:

```julia
julia> function kernel(a)
         a[threadIdx().x] = 0
         return
       end
kernel (generic function with 1 method)

julia> @cuda threads=2 kernel(CuArray([1]))
```

If we execute this code, we'll get a very short error message:

```
ERROR: a exception was thrown during kernel execution.
Run Julia on debug level 2 for device stack traces.
```

As the message suggests, we can have CUDA.jl emit more rich stack trace information by
setting Julia's debug level to 2 or higher by passing `-g2` to the `julia` invocation:

```
ERROR: a exception was thrown during kernel execution.
Stacktrace:
 [1] throw_boundserror at abstractarray.jl:541
 [2] checkbounds at abstractarray.jl:506
 [3] arrayset at /home/tim/Julia/pkg/CUDA/src/device/array.jl:84
 [4] setindex! at /home/tim/Julia/pkg/CUDA/src/device/array.jl:101
 [5] kernel at REPL[4]:2
```

Note that these messages are embedded in the module (CUDA does not support stack unwinding),
and thus bloat its size. To avoid any overhead, you can disable these messages by setting
the debug level to 0 (passing `-g0` to `julia`). This disabled any device-side message, but
retains the host-side detection:

```
julia> @cuda threads=2 kernel(CuArray([1]))
# no device-side error message!

julia> synchronize()
ERROR: KernelException: exception thrown during kernel execution
```


## Debug info and line-number information

Setting the debug level does not only enrich stack traces, it also changes the debug info
emitted in the CUDA module. On debug level 1, which is the default setting if unspecified,
CUDA.jl emits line number information corresponding to `nvcc -lineinfo`. This information
does not hurt performance, and is used by a variety of tools to improve the debugging
experience.

To emit actual debug info as `nvcc -G` does, you need to start Julia on debug level 2 by
passing the flag `-g2`. Support for emitting PTX-compatible debug info is a recent addition
to the NVPTX LLVM back-end, so it's possible this information is incorrect or otherwise
affects compilation.

!!! warning

    Due to bugs in `ptxas`, you need CUDA 11.5 or higher for debug info support.

To disable all debug info emission, start Julia with the flag `-g0`.


## `compute-sanitizer`

To debug kernel issues like memory errors or race conditions, you can use CUDA's `compute-sanitizer` tool. Refer to the [manual](https://docs.nvidia.com/compute-sanitizer/ComputeSanitizer/index.html#using-compute-sanitizer) for more information.

To use `compute-sanitizer`, you need to install the `CUDA_SDK_jll` package in your environment first.

To spawn a new Julia session under `compute-sanitizer`:
```
julia> using CUDA_SDK_jll

# Get location of compute_sanitizer executable
julia> compute_sanitizer = joinpath(CUDA_SDK_jll.artifact_dir, "cuda/compute-sanitizer/compute-sanitizer")
.julia/artifacts/feb6b469b6047f344fec54df2619d65f6b704bdb/cuda/compute-sanitizer/compute-sanitizer

# Recommended options for use with Julia and CUDA.jl
julia> options = ["--launch-timeout=0", "--target-processes=all", "--report-api-errors=no"]
3-element Vector{String}:
 "--launch-timeout=0"
 "--target-processes=all"
 "--report-api-errors=no"

# Run the executable with Julia
julia> run(`$compute_sanitizer $options $(Base.julia_cmd())`)
========= COMPUTE-SANITIZER
julia> using CUDA

julia> CuArray([1]) .+ 1
1-element CuArray{Int64, 1, CUDA.DeviceMemory}:
 2

julia> exit()
========= ERROR SUMMARY: 0 errors
Process(`.julia/artifacts/feb6b469b6047f344fec54df2619d65f6b704bdb/cuda/compute-sanitizer/compute-sanitizer --launch-timeout=0 --target-processes=all --report-api-errors=no julia`, ProcessExited(0))
```

By default, `compute-sanitizer` launches the `memcheck` tool, which is great for dealing with
memory issues. Other tools can be selected with the `--tool` argument, e.g., to find thread
synchronization hazards use `--tool synccheck`, `racecheck` can be used to find shared memory
data races, and `initcheck` is useful for spotting uses of uninitialized device memory.


## `cuda-gdb`

To debug Julia code, you can use the CUDA debugger `cuda-gdb`. When using this tool, it is
recommended to enable Julia debug mode 2 so that debug information is emitted. Do note that
the DWARF info emitted by Julia is currently insufficient to e.g. inspect variables, so the
debug experience will not be pleasant.

If you encounter the `CUDBG_ERROR_UNINITIALIZED` error, ensure all your devices are
supported by `cuda-gdb` (e.g., Kepler-era devices aren't). If some aren't, re-start Julia
with `CUDA_VISIBLE_DEVICES` set to ignore that device.
