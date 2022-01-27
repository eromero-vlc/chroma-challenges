# Short-term Chroma Workflow Challenges

In the following, please find a list of problems that we find running chroma.

## Compiling

- Flags for specific architectures and things: AVX512, OpenMP, CUDA, rocm, gpu intel; special flags for compilers: gcc/clang vs intel; c++20 requirement in QDP-JIT
- Package flavors: QDPXX/QDP-JIT has to be compile with different number of dimensions and precision
- Large number of different packages: QMP, QDPXX/QDP-JIT, QphiX, mgproto, quda, chroma, tensor, adat, colorvec, hadron, harom, laplace\_eigs, superbblas, primme, magma
   - Plus analysis packages: reconfit, itttp
- Lack of maintainability or proper compilation process on some packages: laplace\_eigs, tensor
- BLAS is annoying to setup on cmake software and tensor; cmake is a nightmare in general (leaking abstraction all over the place to ending up putting the compiling/linking flags hard coded, abstracting/hiding CUDA and ROCM support), specially with rocm

### Proposed solutions

- Individual scripts for each tool and platform (`qdp_software`)
- chromaform: poor package manager, automatically download and build and resolve dependencies; hard coded recipes to compile on different platforms and compilers but programmatically organized and explicit
- spark: good luck specifying all the particular cases

## Usage

- For stages complicated enough to have their scripts and internal workflows; and they have a sequential dependency:

  1. Gauge generation (hmc)
  2. Eigenvector generation (chroma single prec, laplace\_eigs)
  3. Distillation objects (baryon, meson, props, genprops): chroma double prec, harom
  4. Corr. func. generation: redstar, colorvec, harom
  5. Analysis

- The job specification is intricate: redstar, chroma and harom take XML documents as input; each task follows a different scheme; the scheme isn't checked (for instance, unused tags aren't reported); no documentation about the meaning of tags or the default values; reading the source code is the way to go

- Flags and env vars for specific architectures; for instance:
  - qphix/mgproto: `-by 2 -bz 2 -pxy 1 -pxyz 1 -c ${OMP_NUM_THREADS} -sy 1 -sz 1`
  - qdp-jit for dist. objs.: `-poolsize 0k  -pool-max-alloc 0  -pool-max-alignment 512`

- Inconsistent or broken internal error management: the codes mixes abort, exit, and exceptions to report errors. The latest isn't properly design for MPI and produces hangs or segmentation faults without reporting the error

- Repeated information and lack of sensible default values: examples:
  - One may choose different smearing for creating the eigenvectors and then for mesons, props, genprops... but usually they are the same
  - The shift and clover term has to be given to the inverters although the information is in the `.lime` file

- Redstar diagram specification seems difficult to use and we ask Robert all the time what to put there

- Minimal support to introspect eigenvectors and distillation objects and check properties; which makes debugging and finding mistakes on the input files cumbersome

- Use of filesystem as a database (encoding object properties in the file's name and path) and no support to organize the generated objects

- Performance limitations of each tool has to be taken into account and limit the number of instances of each task running of the cluster. This is dramatic for generating eigenvectors (solved), genprops and baryons (writing in shared filesystem), and running redstar (limiting the number of open files and reading from shared filesystem)

- No support for checking the success of jobs, and relaunching them, and copy the distillation objects to other facilities

### Proposed solutions

- `chroma_python`: standardized XML tasks generators, including lattice and smearing information
- `https://github.com/eromero-vlc/chroma-scripts-cori`: scripts to create, launch (bundling jobs), check, relaunch, and copy distillation objects away.
  - They have to be customize for each cluster and workflow
- Connect chroma with redstar, and generate on the fly the baryons and mesons (for convenience), and potentially any other distillation object which is to large
- Instead of filehash (sdb) and QDP key-value files (mod), write distillation objects in S3T format (aka superbblas tensor format), which can be easily read in other framework, such as python and matlab
- Extend adat's dbutils to compare distillation objects
- Creation of eigenvectors and mesons on chroma while deprecates laplace\_eigs and avoid writing the gauge field in mod

## Performance

- The multigrid invertors do not strongly scale

- The reading and writing of the genprops from the global filesystem consumes a significant fraction of the total time and quickly saturates the resource as many of the jobs run simultaneously

- Qphix/mgproto are broken; we need an alternative for CPU

- No invertor for intel gpus

- The performance of redstar with threads sucks

- redstar doesn't work with gpus

### Proposed solutions

- Generate genprops on the local disks and immediately run redstar afterwards

- Porting hadron to GPUs

## Analysis

- Ok tools for analysis only for two point functions

### Proposed solutions

- Christos python notebook for 3pt analysis
- Colin C++ 3pt analysis
