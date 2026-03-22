# Performant Matrix Multiplication


## Task 1: Write a Naive Matrix Multiplication

In `src/main.cpp` is code that randomly initializes matrices `A` and `B`.
Write code to multiply the matrices `A` and `B`, forming a new matrix `C`.
Parallelize your naive matrix multiplication code using OpenMP, and include the code in this repository.

Hint: While trying to get this working, you may want to reduce the size of the matrices.

You may use the following code to initialize the matrices:

```c++
#include <iostream>
#include <vector>
#include <random>
#include <cmath>
#include <algorithm>
#include <omp.h>

static const int N = 4096;

std::vector<double> randomMatrix(int n, std::mt19937& rng) {
  std::uniform_real_distribution<double> dist(0.0, 1.0);
  std::vector<double> m(n * n);
  for (auto& v : m)
    v = dist(rng);
  return m;
}

int main(int argc, char *argv[]) {
  std::mt19937 rng(42);

  std::vector<double> A = randomMatrix(N, rng);
  std::vector<double> B = randomMatrix(N, rng);
}
```

## Task 2: Profile the Naive Matrix Multiplicatio

Compute the arithmetic intensity (AI) of your naive matrix multiplication code for matrices of size `4096 x 4096` elements (consult the Getting the Arithmetic Intensity section below for advice on how to do this) when running on 64 threads on a single Perlmutter CPU node.
Include your job submission script in this repository, as well as the full output from your CrayPat experiment.
Report the walltime for the matrix multiplication, as well as the flops, L1 data cache misses, and AI in this `README.md` file.

## Task 3: Write a Tiled Matrix Multiplication

Now, write a tiled matrix multiplication code, with a configurable tile size, and parallelize it using OpenMP.
Include your tiled matrix multiplication source code in this repository.

Hint: In order to do this efficiently, you may need to repack some of the data associated with each tile into a contiguous buffer.  In your loops over tiles, whenever you start a new tile, consider copying the values of matrices `A` or `B` associated with that tile into some appropriate temporary space that packs the data contiguously.  Then use the packed buffer when you need to access data from the tile.  Similarly, you can reserve temporary buffer space for a contiguous tile of matrix `C`, and only update `C` itself after the full tile has been computed.

Note: When running with matrices of `4096 x 4096` elements and on 64 OpenMP threads, your code should be reasonably well load balanced up to a tile size of at least `512 x 512` elements.

## Task 4: Profile the Tiled Matrix Multiplication

Compute the arithmetic intensity (AI) of your tiled matrix multiplication code for matrices of size `4096 x 4096` (consult the Getting the Arithmetic Intensity section below for advice on how to do this) when running on 64 threads on a single Perlmutter CPU node.
Perform this test for tile sizes of `8 x 8`, `16 x 16`, `32 x 32`, `64 x 64`, `96 x 96`, `128 x 128`, `256 x 256`, and `512 x 512` elements, including the full output from each of the CrayPat experiments.

Create a plot of the walltime for the matrix multiplication vs. tile size, as well as a plot of the L1 data cache misses vs. tile size.
Place both plots in this repository.

## Task 5 Explain the Results

Compare the results from the naive matrix multiplication with the results from the tiled matrix multiplication.
Explain any trends you observe in the plots in Task 4.









## Using Perlmutter

Note that you will need to login to Perlmutter to complete this problem.
You may find NERSC's [getting started](https://docs.nersc.gov/getting-started/) documentation helpful for this purpose.
If you have a working account, you can do this by executing the following on Mac or Linux:

```bash
ssh <username>@perlmutter.nersc.gov
```

Then enter your password as `<password><MFA code>`.
There shouldn't be any whitespace between your normal password and the generated multi-factor authentication code.
On Windows, you may find it useful to use [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) or a similar tool to login.

You will need to be able to access you code on Perlmutter.
You can do this by either configuring your Perlmutter environment to work with GitHub, or by copying files with SCP.
For example, to copy a directory onto Perlmutter, you can do the following from your local machine:

```bash
scp -r <path_to_directory> <username>@perlmutter.nersc.gov:<path_to_destination>
```

Then enter your password and MFA as before.
To copy back, you can do the following from your local machine:


```bash
scp -r <username>@perlmutter.nersc.gov:<path_to_directory> <path_to_destination>
```



## Getting the Arithmetic Intensity

To get the arithmetic intensity on a CPU node, you can use [CrayPat](https://docs.nersc.gov/tools/performance/craypat/).
In order to use CrayPat, you'll need to build an instrumented executable that supports CrayPat.
Aside from the standard NERSC documentation, you may consider the following lines (note that you are not required to use CMake for this problem).

```bash
module unload darshan
module load PrgEnv-gnu
module load perftools-base perftools

# Build the code
cd src
rm main.o main main+pat
CC -fopenmp -c main.cpp
CC -fopenmp -o main main.o
pat_build main
```

Note that it is **very** important to have `perftools` loaded before compiling.
If everything works correctly, you should get an executable named `main+pat` (or `<executable_name>+pat`), which you can run to collect analytics.
You'll need to set the `PAT_RT_PERFCTR` environment variable to indicate which hardware counters to monitor during the experiment.
In particular, we'll want to know the number of floating point operations (`PAPI_FP_OPS`) and the number of L1 data cache misses (`PAPI_L1_DCM`).
The latter gives us a decent sense of our DRAM memory bandwidth usage.
To run, you can consider the following lines:

```
module unload darshan
module load PrgEnv-gnu
module load perftools-base perftools

cd src

# Remove any prior CrayPat output directory
rm -rf pat_ai

# Configure CrayPat to give use the data we want
export PAT_RT_CALLSTACK_MODE=frames
export PAT_RT_EXPDIR_NAME=pat_ai
export PAT_RT_PERFCTR="PAPI_FP_OPS,PAPI_L1_DCM,PAPI_L2_DCM"

export OMP_PROC_BIND=spread
export OMP_PLACES=threads
export OMP_NUM_THREADS=64

# Run the code
srun -n 1 -c 128 --cpu_bind=cores ./main+pat

# Get the report
pat_report -O hwpc pat_ai/*
```

You should be able to find the values of `PAPI_FP_OPS` and `PAPI_L1_DCM` in the report.


## Answers

