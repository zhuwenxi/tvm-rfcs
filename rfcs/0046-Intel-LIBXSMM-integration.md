# Summary
This RFC introduces the plan of integrating LIBXSMM into TVM. LIBXSMM leverages JIT code generator to produce high efficient kernels targeting x86 architectures. 

For details of LIBXSMM, please refer to:
* [LIBXSMM User Manual](https://libxsmm.readthedocs.io/en/latest/)
* [LIBXSMM github repo](https://github.com/hfp/libxsmm)

# Motivation
TVM has shown satisfactory performance on MLP models with CPU. However there are still some defects in the assembly code generated by LLVM which block AutoTVM/AutoScheduler from achieving optimal on GEMM.

LIBXSMM is a open source library developed by Intel Lab for accelerating small matrix multiplication. It leverages the JIT code generator to generate high efficient GEMM kernels for x86 CPU, which could be very close to hardware rootline. According to our evaluation, in “small” GEMM (cube_root(m * n * k) <= 256) , LIBXSMM shows a superior performance over the well-known BLAS library Intel MKL. 

By the way, given that LIBXSMM can generate quite efficient GEMM kernel implementation, it is also an ideal substitution for inner-kernel of normal size GEMM. According our experiments, the AutoTVM templates we wrote with LIBXSMM as register-block generation, has a much higher performance comparing to MKL and existing TOPI implementation.

# Guide-level explanation
This proposal aims to integrate LIBXSMM into TVM to accelerate small GEMM and serve as inner-kernel to accelerate normal size GEMM.

We will integrate LIBXSMM with TVM in following 3 components:
1. Add extern call “tvm.contrib.libxsmm.gemm” in “src/runtime/contrib” directory, and corresponding python interface in "python/tvm/contrib/" directory, so users can call them just as CBLAS; 
2. Use BYOC to accelerate small GEMM (cube_root(m * n * k ) <= 256) and its epilogue fusion variations (bias/relu/sigmoid/bias_relu/bias_sigmoid);
3. AutoTVM template we wrote with LIBXSMM as inner kernel into TOPI, as a GEMM implementation candidate.

# Reference-level explanation
1. Users can call libxsmm as CBLAS through extern call API.
```python
  def matmul(lhs, rhs, transa=False, transb=False, alpha=1.0, beta=0.0, lda=-1, ldb=-1, ldc=-1, **kwargs):
    n = lhs.shape[1] if transa else lhs.shape[0]
    m = rhs.shape[0] if transb else rhs.shape[1]
    return te.extern(
      (n, m),
      [lhs, rhs],
      lambda ins, outs: tvm.tir.call_packed(
        "tvm.contrib.libxsmm.matmul", ins[0], ins[1], outs[0], transa, transb, alpha, beta, lda, ldb, ldc),
      name="C",
      **kwargs,
  )
```
2. BYOC allows for graph partitioning and using LIBXSMM for code generation.
  * API to obtain the partitioned function:
```python
  from tvm.relay.op.contrib import libxsmm

  # API to call LIBXSMM partitioning
  libxsmm_module = libxsmm.partition_for_libxsmm(module) 
```
  * Pattern matching table: 
```python
  @register_pattern_table("libxsmm")
  def pattern_table():
      dense_pattern = ("libxsmm.dense", make_pattern(with_bias=False, with_activation=None))
      denese_bias_pattern = ("libxsmm.dense_bias", make_pattern(with_bias=True, with_activation=None))
      denese_relu_pattern = ("libxsmm.dense_relu", make_pattern(with_bias=False, with_activation="relu"))
      denese_sigmoid_pattern = ("libxsmm.dense_sigmoid", make_pattern(with_bias=False, with_activation="sigmoid"))
      denese_bias_relu = ("libxsmm.dense_bias_relu", make_pattern(with_bias=True, with_activation="relu"))
      denese_bias_sigmoid = ("libxsmm.dense_bias_sigmoid", make_pattern(with_bias=True, with_activation="sigmoid"))
      libxsmm_pattern = [dense_pattern, denese_bias_pattern, denese_relu_pattern, denese_sigmoid_pattern, denese_bias_relu, denese_bias_sigmoid]
      return libxsmm_pattern
```
  * Build with TVM
```python
  with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(libxsmm_module, target="cpu", params=params)
```
3. Integrate into TOPI, an GEMM autotvm template with LIBXSMM as inner kernel.
  * Use Tensorize/TensorIR to substitute register block of GEMM with LIBXSMM
```python
  def intrin_func(ins, outs):
    def _body():
      ib = tvm.tir.ir_builder.create()
      ib.emit(
        tvm.tir.call_extern(
          "int", "libxsmm_sgemm", m, n, k, 1.0, ins[0].access_ptr("r"), K, ins[1].access_ptr("r"), n, 0.0, outs[0].access_ptr("w"), N
        )
      )
      return ib.get()

    def _update():
      ib = tvm.tir.ir_builder.create()
      ib.emit(
        tvm.tir.call_extern(
           "int", "libxsmm_sgemm", m, n, k, 1.0, ins[0].access_ptr("r"), K, ins[1].access_ptr("r"), n, 1.0, outs[0].access_ptr("w"), N
        )
      )
      return ib.get()
```

# Testing
We will add unittest for coresponding extern call, BYOC and TOPI related code:
* Make sure the result LIBXSMM produces is correct with its TVM counter part;
* Confirm match patterns are working as expected.

# Drawbacks
* Though LIBXSMM works well with AutoTVM, it does not help AutoScheduler;
* Memory footprint would increase as JIT code generated, a LRU kernel cache might be required to mitigate it. 

# Future possibilities
* LIBXSMM has DNN support, so it might be interesting to also integrate DNN primitives such as Conv to TVM;
* LIBXSMM has quantized kernel (int8), we can also integrate it to TVM, as long as it surpass existing oneDNN implementations.