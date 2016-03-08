rsgdx-mini
=============

* version: 1.0 (based on RSGDX obtained on 2014/02/13)
* date: 2016/03/08
* contact: miniapp@riken.jp


About RSGDX abd rsgdx-mini
--------------------------------
RSGDX is the application software that computes the time-space variation of
the slip faulting caused by the tectonic stress and fault friction, aiming to
simulate the cycle of earthquake occurrences.
The analysis data size for RSGDX ranges from 10,000 cells to 1,000,000 cells
requiring the input data ranging from Gigabytes to Terabytes.

rsgdx-mini is a small packaged RSGDX prepared as a member of mini-appli suits.
It contains the source files, small size test data with 1236 cells,
and installation scripts.

The parallel program model used in rsgdx-mini is MPI and autoparallel.
Each of the rsgdx-mini MPI processes contains the cell information of:
 - matrix representing the mutual interaction with other cell elements
 - vector representing the slip velocity of the cell
Each MPI process can take advantage of autoparallel feature of compilers,
usually applied to the nested loops for matrices computations.
Data communication is done mostly using MPI_Allgather() and MPI_Allreduce().


Installation
------------

The installation steps of rsgdx-mini on Linux platform are explained below.
Confirm that the prerequisite software is available on the platform.
Also plan on the directory location where the package should be installed.

+ Prerequisite software:
  - Fortran and C compiler
  - MPI library,
  - BLAS library (ddot/daxpy/dgemv)

+ directory planning
  - program installation top directory
  - job execution top direcctory

####step 1. Obtain the distribution package.

Obtain the rsgdx-mini package file "rsgdx-mini-1.0.tar.gz" from the repository.
+ http://fiber-miniapp.github.io/

On the appropriate directory, extract the contents using tar command.

    $ tar -zxf rsgdx-mini-1.0.tar.gz
    $ cd ./rsgdx-mini
	$ TOP_DIR=${PWD}

The variable TOP_DIR is used hereafter to point the installation directory.
This readme file (README.md) is included in the ${TOP_DIR} directory.


####step 2.  Choose Makefile for the installing platform.

In src/ directory, there are several Makefile.* files for different platforms.
Choose one of these Makefile.* , and copy it as Makefile.

    $ cd ${TOP_DIR}/src
    $ ls Makefile.*
	Makefile    Makefile.Linux64-intel-impi  Makefile.fx10
	Makefile.K  Makefile.Linux64-pgi-ompi


If the installing platform supports module command and modulefiles for choosing
the programming environment, it is recommended to use them for easier setup.
Compiling time of the source program should be short, i.e. within a minute.
After the successful compilation, the executable file "rsgdx.ex" should be
created in the same directory.

##### Example: Installing on Intel platform
Below is an example of compiling with Intel compiler, Intel MPI and Intel MKL
with module environment.
If the platform does not support modulefiles, then set the appropriate
environment variables such as PATH and LD_LIBRARY_PATH needed for the compiler
software.
Typical settings are commented inside of Makefile.* files.

    $ cp Makefile.Linux64-intel-impi Makefile
    $ module load intel impi
    $ make
    $ ls -go rsgdx.ex
	-rwxr-xr-x 1 1293176 Sep 22 12:24 rsgdx.ex

##### Example: Installing on K computer
Below is an example of compiling on K computer.

    $ cp Makefile.K Makefile
    $ make
    $ ls -go rsgdx.ex
	-rwxr-xr-x 1 1140539 Sept 22 13:35 2014 rsgdx.ex



Running a test job
--------------------
Running a test job will need two step execution.
- step 1, preprocess the supermatrix data according to the number of MPIs the
  job will use.
- step 2, run the test job using the input data files created in the first step.

####step 1.  Preprocess the supermatrix data

A source file named "partition_supermatrix.f90" 
is included under partition/ directory.
And a small model data file named "supermatrix.dat" is included under
data/mini_1Ddiv directory.
In order to create the input files for various number of MPI processes,
the "supermatrix.dat" file must be preprocessed according to the number of MPIs
for the job. A standalone program "partition_supermatrix.f90" is provided
to do this.
First, change the line 28 of "partition_supermatrix.f90", and set the
value of NPROC to the number of MPIs the job will use.

      integer(KP), parameter :: NPROC   =      64

Then compile and run the program with the input "supermatrix.dat" file.
The "supermatrix.dat" file is now decomposed into the domain data files
and the boundary data files.
Below is an example of creating the input files for 8 processes.

	$ cd ${TOP_DIR}/partition
	$ NPROCS=8
	$ sed  "28s/  64/${NPROCS}/" ./partition_supermatrix.f90 > main.f90
	$ ifort -o main-${NPROCS}.ex main.f90
	$
	$ mkdir -p ${TOP_DIR}/subs_${NPROCS}
	$ cd       ${TOP_DIR}/subs_${NPROCS}
	$ ln -s ${TOP_DIR}/data/mini_1Ddiv/supermatrix.dat supermatrix.dat
	$ ${TOP_DIR}/partition/main-${NPROCS}.ex

After the preprocessing, the following files should reside on the
current working directory.

	$ ls
	constant.mod   dfullmat.0006  drkmat.0005    operation_conunt_etc.out
	dfullmat.0000  dfullmat.0007  drkmat.0006    precisions.mod
	dfullmat.0001  drkmat.0000    drkmat.0007    supermatrix.dat
	dfullmat.0002  drkmat.0001    flops.mod      util.mod
	dfullmat.0003  drkmat.0002    hmatrices.mod
	dfullmat.0004  drkmat.0003    main-8.ex
	dfullmat.0005  drkmat.0004    main.f90

Note that preprocessing the supermatrix data is done by a single process
executable "main-${NPROCS}.ex". The data included in the data/mini_1Ddiv
is small and can be preprocessed on any system.
The preprocessing the large data will require memory size exceeding 16 GB,
and is often done on a separate server that has enough memory size.


####step 2. Run the test job using the created data files

To run a test job using the input data files created in the previous step,
copyin the created data files and the parameter files to the
working directory.
The following data files must exist before the simulation starts.
+ in the current working directory:
 - input_timer.nml
 - inputpara.g_000.nml
+ in the subdirectory named 000/:
 - dfullmat.*
 - drkmat.*
 - hlib_a_b.p_all

Below is an example of running 8 process job.
##### Example: Running the test job on Intel platform using 8 process

	$ NPROCS=8
	$ cd ${TOP_DIR}/subs_${NPROCS}
	$ if [ $? != 0 ] ; then echo '@@@ Directory error @@@'; exit; fi
	$ mkdir 000
	$ mv dfullmat.* drkmat.* 000/
	$ cp -p ${TOP_DIR}/partition/hlib_a_b.p_all 000/
	$ cp -p ${TOP_DIR}/partition/input_timer.nml .
	$ cp -p ${TOP_DIR}/partition/inputpara.g_000.nml .
	$ export OMP_NUM_THREADS=8
	$ time -p mpirun -np 8 ${TOP_DIR}/src/rsgdx-${NPROCS}.ex


##### Example: Running the test job on K computer using 8 process

If the running platform requires the file staging, such as on K computer,
the following files must be staged.
+ list of files and directories to be staged
 - stage-in-dir: ${TOP_DIR}/src		as src
 - stage-in-dir: ${TOP_DIR}/subs_${NPROCS}		as subs_${NPROCS}
 - stage-in-dir: ${TOP_DIR}/subs_${NPROCS}/000	as subs_${NPROCS}/000
 - stage-out-file ./monitor_000		# monitor text file
 - stage-out-file ./000/*_output.p_*	# optional binary output

An example job script for K computer is shown below.
Note that the variable TOP_DIR must be substituted with the actual
value before submitting the job.

	#PJM -N RSGDX_RUN_K8
	#PJM --rsc-list "elapse=01:00:00"
	#PJM --rsc-list "node=8"
	#PJM --mpi "proc=8"
	#PJM --mpi "use-rankdir"
	#PJM --stg-transfiles all
	#PJM --stgin-basedir "${TOP_DIR}/subs_8"
	#PJM --stgin "rank=* "${TOP_DIR}/src/rsgdx-8.ex  %r:./ "
	#PJM --stgin "rank=* ./input_timer.nml  %r:./ "
	#PJM --stgin "rank=* ./inputpara.g_000.nml  %r:./ "
	#PJM --stgin "rank=* ./000/*.%04r  %r:000/ "
	#PJM --stgin "rank=0 ./000/hlib_a_b.p_all  ./000/ "
	#PJM --stgout-basedir "${TOP_DIR}/logs_K"
	#PJM --stgout "rank=* %r:./000/*_output.p_* ./ "
	#PJM --stgout "rank=0 ./monitor_000  ./ "
	#PJM -j
	#PJM -S
	source /work/system/Env_base
	NPROCS=8
	export OMP_NUM_THREADS=8
	time -p mpiexec -n ${NPROCS} ./rsgdx-${NPROCS}.ex -Wl,-T12


###Verifying the computed results (This section must be updated)

In the current working directory, the file "monitor_000" contains the
time step information for major variable.
Observe that the values are not diverging.
Below is a reference example of the "monitor_000" file for 64 MPI job.

	$ cat monitor_000 
	k=    0  0.000000E+00  0.000000E+00  0.000000E+00
	k=    1  0.000000E+00  0.000000E+00  0.100000E+04
	k=    2  0.316887E-04  0.100000E+04  0.500000E+04
	k=    3  0.190132E-03  0.500000E+04  0.250000E+05
	k=    4  0.982351E-03  0.250000E+05  0.125000E+06
	k=    5  0.494344E-02  0.125000E+06  0.625000E+06
	k=    6  0.247489E-01  0.625000E+06  0.265103E+07
	k=    7  0.108757E+00  0.265103E+07  0.261581E+07
	k=    8  0.191648E+00  0.261581E+07  0.309013E+07
	k=    9  0.289571E+00  0.309013E+07  0.366240E+07


Running the scalability tests with large data set
-----------------------------
In order to run the scalability tests, a separate data set large enough to
provide the computing volume should be used.
Such large data set is not contained in this rsgdx-mini, but can be provided
per request base by writing to the contact point whose address is shown at
the top of thie README file.
The large data set file "supermatrix.dat" which contains 1228800 surface
elements can be provided upon such request.
With this file, the necessary input data files for the scalability tests
can be generated just as explained in the previous section,
i.e. step 1.  Preprocess the supermatrix data.
The time to generage the input data files from "supermatrix.dat" will often
take days.
For the case of NPROCS=1024, the size of total input files is 1.4 TB.
Theoretically, the job can be run with up to 1228800 MPIs, i.e. 2^14 x 5^2 x 3,
even though such scale test has not been carried out.


License term (This section must be updated)
-----------------------------
The License term of RSGDX is provided in the BSD 2-Clause License.
Please refer to "LICENSE" file included in the rsgdx-mini package.

Contact for inquiry regarding rsgdx-mini, see the top of this README.

