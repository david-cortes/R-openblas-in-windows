# Using OpenBLAS in R for Windows

This is a short post illustrating how to easily get R for Windows to use OpenBLAS as the backend for linear algebra operations.

Most tutorials out there are about building R itself with OpenBLAS from source which is very complex, but this is not necessary as long as one is willing to make the installation some 30-60mb heavier, which should not be a problem with today's hard disks.

### What is OpenBLAS?

[OpenBLAS](https://www.openblas.net) is a [BLAS](https://en.wikipedia.org/wiki/Basic_Linear_Algebra_Subprograms) (basic linear algebra sub-routines) and [LAPACK](https://en.wikipedia.org/wiki/LAPACK) (linear algebra package) backend, for performing fast linear algebra operations - examples: matrix multiplications, solving linear systems, calculating eigenvalues, among many others. These sound like trivial operations, but their execution speed can vary a lot across different software, so the specific backend used for them matters.

By default, R for Windows ships its own un-optimized BLAS and LAPACK replacements, which do the job in the sense that they offer all the necessary functionality (for R) and compute the results correctly, but are much slower than optimized BLASes as they do not exploit all the features of modern CPUs.

### What difference does it make?

This is a time comparison of a simple operation (matrix multiplication) with the default R BLAS and with OpenBLAS:
```r
library(microbenchmark)
set.seed(1)
m <- 1e4
n <- 1e3
k <- 3e2
X <- matrix(rnorm(m*k), nrow=m)
Y <- matrix(rnorm(n*k), ncol=n)
microbenchmark({
    Z <- X %*% Y
}, times=10L)
```

When ran on my own setup (AMD Ryzen 7 2700, 8c/16t, 3.2GHz), I obtain the following timings:

* With R's BLAS:
```
Unit: seconds
                 expr      min       lq     mean   median      uq      max neval
 {     Z <- X %*% Y } 2.035071 2.060824 2.072882 2.069541 2.09677 2.104811    10
```

* With OpenBLAS:
```
Unit: milliseconds
                 expr     min      lq     mean  median       uq      max neval
 {     Z <- X %*% Y } 58.6489 71.2553 83.42615 72.7103 104.9017 110.6942    10
```

That is to say: using OpenBLAS made the operation almost 30x faster. No changes in the code were required, and it will speed up operations from CRAN packages which rely on BLAS too.


### What you will need

The following will be required in order to follow the next steps:
* Write access to the folder where R is installed. If R was installed with the default options in the installer, saying 'Yes' to all, this will mean that administrator rights will be required.
* Some compression/de-compression software that could extract compressed files from Python wheels, such as [7-zip](https://www.7-zip.org).

### Instructions in detail

* Download the [Numpy wheel for Windows](https://pypi.org/project/numpy/#files) from PyPI. There are many of them so it needs to be the correct variant: should say `win` (as this is for windows), and should match to the computer architecture (`amd64` for 64-bit windows versions, `win32` for 32-bit version).
* De-compress (un-zip) the wheel. If using 7-zip, this can be done by right-clicking the file, selecting '7-zip' and then 'Extract to ...' or similar.
* In the folder where the contents of the wheel archive were extracted, locate some file ending in `.dll` with a name containing `openblas`. Typically, this should be under `<folder>\numpy\.libs\`, and might be named as something like `libopenblas.GK7GX5KEQ4F6UYO3P26ULGBQYHGQO7J4.gfortran-win_amd64.dll`. Keep this file at hand for later.
* Locate the folder where R itself is installed. Typically, this should be something like: `C:\Program Files\R\R-4.0.5` (or some other version depending on what you have installed).
* Within the R folder, locate the sub-folder `bin\x64` (e.g. C:\Program Files\R\R-4.0.5\bin\x64).
* In this folder there should be two key files: `Rblas.dll` and `Rlapack.dll`. Copy them somewhere to have a backup if anything goes wrong.
* Delete these two files (`Rblas.dll` and `Rlapack.dll`) from `bin\x64`.
* Copy the openblas dll file which was extracted from the NumPy wheel (e.g. `libopenblas.GK7GX5KEQ4F6UYO3P26ULGBQYHGQO7J4.gfortran-win_amd64.dll`) to this same folder **twice**.
* Rename one of the copies as `Rblas.dll` and the other as `Rlapack.dll`. Hint: under the default window settings, file extensions will be hidden, in which case the `.dll` part should be left ot when renaming them.
* Optionally, do the same process for the folder `i386` which should be at the same level as `x64` (e.g. `C:\Program Files\R\R-4.0.5\bin\i386`).

At this point you're done and the next time you start R it will already be using OpenBLAS for accelerated linear algebra operations.

### To keep in mind

The OpenBLAS that was taken from NumPy supports multi-threading, which is controlled through an environment variable `OPENBLAS_NUM_THREADS`. Environment variables can be set through the control panel in windows. Alternatively, they can be set in R itself through e.g. `Sys.setenv("OPENBLAS_NUM_THREADS" = as.character(parallel::detectCores()))`, but setting the threads after R has already started might not have any effect. As yet another alternative, the number of threads can be set through the package `RhpcBLASctl`.
