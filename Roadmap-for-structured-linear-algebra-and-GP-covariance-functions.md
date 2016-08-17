### Introduction

Stan's linear algebra functions assume dense matrices with no special structure. We intend to add specialized matrix types in order to speed up matrix computations when matrices adhere to specific structures such as symmetric positive definite, Toeplitz, banded, and sparse matrices. As the new types are added, we will specialize the GP covariance functions as well to return the appropriate structured matrix type based on user specifications. We do not plan to do automated detection of specialized structure.

### Roadmap

The current roadmap is as follows:

* Implement symmetric, positive definite Toeplitz type (can be banded SPD Toeplitz)
* Specialize addition, matrix multiplication for SPD Toeplitz type
* Specialize Cholesky and inverse for SPD Toeplitz type
* Specialize log determinant for SPD Toeplitz
5. Specialize cov_exp_quad to return SPD Toeplitz
6. Add Kronecker and Hadamard products for arbitrary matrices
7 Implement SPD type
8 Specialize cov_exp_quad to return SPD
9 Specialize Eigendecomposition for SPD matrices
10 Implement sparse matrix type
12 Implement SPD sparse matrix type
13 Specialize matrix algebra functions for sparse matrix type
14 Specialize matrix algebra functions for SPD sparse matrix type
15 Specialize decompositions for sparse matrix type
16 Specialize decompositions for SPD sparse matrix type





###