---
title: "通用矩阵乘法并行优化 / Parallel Optimization on General Matrix Multiplication"
categories:
    - HPC
tags:
    - HPC
---

## 中文版本
这段时间在准备托福，所以中文版本先放放，过段时间再补（或者不补，看我的心情）。如果你也在准备雅思/托福/GRE/四六级，你也可以尝试阅读下面的英文版本。其实对于我来说，某种程度上写这些技术类的博客，英文会更顺手一点（bushi

## English Version
It has been a long time since the last blog was released. Recently, however, I saw some friend's blog and was fascinated by his articles, and decided to pick up blogging again. 

In this blog, I would provide some approaches for parallel optimization on general matrix multiplication. These approaches are not meant to optimize the algorithm itself or propose a new algorithm to reduce the time complexity, but just to parallelize it. General matrix multiplication, in short, GEMM, is an algorithm that multiplies two matrices without restrictions to their shapes, which is widely used in scientific computing and machine learning. The pseudo-code of the naïve algorithm is as follows:
```
function GEMM(A, B, Out, M, K, N)
    for i ← 1 to M
        for j ← 1 to N
            for k ← 1 to K
                Out[i][j] ← Out[i][j] + A[i][k] * B[k][j]
            end for
        end for
    end for
end function
```
The data types of parameters depends on the usage of the function. If it is for LINPACK benchmarking, it should be double. And if it is for machine learning, it is usually fp16 or bf16. 

The form of the matrices also matters in the implementation. In Fortran, two-dimensional arrays are usually column-majored, while in C, these are usually row-majored. 

To make life easier, I will use C languages in the following implementations, and use double for the data type of the elements. Here, data will be stored in one-dimensional arrays rather than arrays of arrays or built-in two-dimensional arrays. Here is the C equivalent of the previous pseudo code:
```c
void GEMM(double *A, double *B, double *Out, int M, int K, int N) {
    for (int i = 0; i < M; ++i) {
        for (int j = 0; j < N; ++j) {
            for (int k = 0; k < K; ++k) {
                Out[i * N + j] += A[i * K + k] * B[k * N + j];
            }
        }
    }
}
```

Next, I will propose some "magics" to optimize it without changing the algorithm.

### Memory Locality
Some may thought that this part is not related to parallel computing. They are actually wrong. Improving the memory locality not only better utilizes caches, but also allows the compiler to auto-vectorize code better to achieve data parallelism. Therefore, improving the memory locality might be the easier way to improve the performance. 

In this example, we can firstly observe that, the order of loops does not matter. That is, exchanging the orders of loops will not change the outcome. Therefore, if we change the "j" loop and "k" loop, `A[i * K + k]` will be loaded to the register only once per "k" loop, and both `B` and `Out` elements in "j" loops are adjacent to each others. Therefore, it improves the memory locality, and, if "-O3" is turned on for the compiler, it can possibly auto-vectorize the inner loop. Here is the modified program.

```c
void dgemm(double *A, double *B, double *Out, int M, int K, int N) {
    for (int i = 0; i < M; ++i) {
        for (int k = 0; k < K; ++k) {
            for (int j = 0; j < N; ++j) {
                Out[i * N + j] += A[i * K + k] * B[k * N + j];
            }
        }
    }
}

```

Also, in some compilers, using `__builtin_prefetch` function may improve the performance of memory loads by hinting the processor to prefetch the elements at a specific address. However, modern compilers are so smart that you may not need to do that. 

### Vector/SIMD
What if the compiler is not smart enough to auto-vectorize loops? You have to use intrinsics to manually vectorize them. This could be significant on x86_64 processors where some compilers are extremely conservative on AVX512 and FMA even if you hint them to use these instructions.

Here, we vectorize the inner loop with AVX512 and FMA for better performance. AVX512 utilizes ZMM registers, which are of 512-bit length. A double precision floating-point number is 64-bit long, so a ZMM register can contain 8 double numbers. From the observation above, we know that `A[i * K + k]` would not change during the inner loop, so we broadcast this element to all slots of a vector register, that is, use `_mm512_set1_pd` function. Next, iterate 8 elements in a roll. Use `_mm512_loadu_pd` to load B and Out into vector registers. Note that this function is for non-aligned loading, so it should be applicable for any inputs (but if the input data can be ensured aligned, using `_mm512_load_pd` would increase the performance). Next, use `_mm512_fmadd_pd` to compute the result, and use `_mm512_storeu_pd` to store the result back to the memory.

For the rest part, if j is less than N, it means that some elements are not processed. Therefore, one common way is to iterate all the elements and calculate each of them one by one. However, for AVX512, there is an alternative way to do that: masking. Compute the mask with `(1 << (N - j)) - 1` to mask the elements that should not be loaded or stored. Then replace all the load and store operations with masked load and store. Therefore, it will complete the rest computations without looping.
Here is the complete code with AVX512 and FMA:
```c
#include <immintrin.h>
void dgemm(double *A, double *B, double *Out, size_t M, size_t K, size_t N) {
    for (int i = 0; i < M; ++i) {
        for (int k = 0; k < K; ++k) {
            __m512d a = _mm512_set1_pd(A[i * K + k]);
            int j = 0;
            for (; j + 7 < N; j += 8) {
                __m512d b = _mm512_loadu_pd(&B[k * N + j]);
                __m512d out = _mm512_loadu_pd(&Out[i * N + j]);
                _mm512_storeu_pd(&Out[i * N + j],
                    _mm512_fmadd_pd(a, b, out));
            }
            if (j < N) {
                unsigned char mask = (1 << (N - j)) - 1;
                __m512d b = _mm512_maskz_loadu_pd(mask, &B[k * N + j]);
                __m512d out = _mm512_maskz_loadu_pd(mask, &Out[i * N + j]);
                _mm512_mask_storeu_pd(&Out[i * N + j], mask, 
                    _mm512_maskz_fmadd_pd(mask, a, b, out));
            }
        }
    }
}
```
Here is an ARM NEON version of the above code:
```c
#include <arm_neon.h>
void dgemm(double *A, double *B, double *Out, size_t M, size_t K, size_t N) {
    for (int i = 0; i < M; ++i) {
        for (int k = 0; k < K; ++k) {
            float64x2_t a = vdupq_n_f64(A[i * K + k]);
            int j = 0;
            for (; j + 1 < N; j += 2) {
                float64x2_t b = vld1q_f64(&B[k * N + j]);
                float64x2_t out = vld1q_f64(&Out[i * N + j]);
                vst1q_f64(&Out[i * N + j],
                    vfmaq_f64(out, a, b));
            }
            if (j < N) {
                Out[i * N + j] += A[i * K + k] * B[k * N + j];
            }
        }
    }
}
```
Since ARM NEON only has 128-bit vectors, it can only process two elements once and its performance could be significantly lower than AVX512. On Apple Silicon Mac with Apple Clang, the performance could even be lower than the naïve version with "-O3" optimization. Therefore, this solution only provides a proof-of-concept of this, but no practical use. Maybe it would be better for SVE and SVE2 on ARMv9, but before I got a device with them, I will keep it anyway.

### Multithreading -- OpenMP


### Distributed Computing -- MPI

### Heterogeneous Computing -- OpenACC