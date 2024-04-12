In many cases when a program crashes it produces a core dump file. If the program was compiled with debug information, the core dump file can provide a lot of useful information about the cause of the crash such as the call stack at the moment of the crash.

# 1. Accessing the core dump file

In the old Linux systems the core dump file was produced in the working directory. However, in modern systems there are services that handle the creation of core dump files. You can find many details about the underlying systems in the [section about core dumps of the Arch wiki](https://wiki.archlinux.org/title/Core_dump) [1].

## 1.1 Enabling the creation of core dump files

In modern systems there is control over the resources used for the storage of core dump files, and in specifically there are limits in the size of the file. By default the limit is zero which means that core dump files are not stored. You can check the current value of the maximum core dump size with the command
```
ulimit -c
```
which print `0` if default settings are used. To enable saving core dump files of unlimited size use the command:
```bash
ulimit -c unlimited
```

The `ulimit` built-in command of the shell controls the resources available to the shell and to the processes started by the shell. The resource limits are set to their default values every time you start a new shell.

## 1.2 Locating the core dump files

Coredump files are stored in `/var/lib/systemd/coredump` by default. Find the coredump file in `/var/lib/systemd/coredump` and copy it to a local directory. Usually core dump files have a descriptive name, such as
```
core.python.490151545.6821193296194d308b6e51962e49af48.954748.1712836768000000.lz4
``` 
for a crash of the Python interpreter (`python`). Be reasonably fast because the `/var/lib/systemd/coredump` is periodically cleaned by systemd.  There are options to control the default storage location [1] but they require root privileges and they cannot be used in HPC clusters.

In your *local system* the `coredumpctl` program can retrieve a lot of useful information from the system journal without having to save the core dump files. Simply run
```bash
coredumpctl list
```
to get a list of core dump events, with the `PID` of the process that caused the core dump, and
```
coredumpctl info <PID>
```
to get information that the system journal saved about the core dump event, such as the call stack.

## 1.3 Decompressing the core dump files

When you set the `ulimit -c` to something greater than `0` or `unlimited`, core dump files will be saved in `/var/lib/systemd/coredump` in compressed format. The files are compressed using LZ4. You can decompress them using the `unlz4` command. If the `unlz4` is not available in your system, you can install the [`conda-forge::lz4-c`](https://anaconda.org/conda-forge/lz4-c) Conda package, from the [conda-forge channel](https://conda-forge.org/), that provides `unlz4`.

# 2. Reading the core dump file with GBD

A raw core dump file must be read with GDB to get any useful information regarding the cause of the crash. To to that you need access to the uncompressed core dump file, and the executable and dynamically linked libraries that were involved in the crash. Both the executable and the libraries must be compiled with debug symbols, so that the call stack can be populated with a series of function names instead of simply listing memory locations.

## 2.1 Opening the core dump file

Decompress the core dump with LZ4. For instance, for a crash of the Python interpreter (`python`),
```bash
unlz4 /var/lib/systemd/coredump/core.python.490151545.6821193296194d308b6e51962e49af48.954748.1712836768000000.lz4 -d ./python.coredump
```
will decompress the core dump in `./python.coredump`.

Run the program you executed when the core dump was created with GDB and provide the core dump with the `-d` flag:
```
gdb python -c python.coredump
```
No need to provide inputs, these are saved in the coredump file. Make sure you are in the correct environment! For instance if the core dump was produces while running Python in some Conda environment  `pyscipopt`, run GDB in the same Conda environment so that GDB has access to the Python executable and libraries mentioned in the core dump, otherwise GBD will produce nonsense.

## 2.2 Reading the GBD output

When starting, GDB will print the top of the stack where the core was dumped. For instance, in our example GDB produced:
```
(pyscipopt) $ gdb python -c python.coredump
GNU gdb (GDB) Red Hat Enterprise Linux 8.2-20.el8
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from python...done.

warning: core file may not match specified executable file.
[New LWP 954748]
BFD: warning: /mnt/irisgpfs/users/gmaudet/micromamba/envs/pyscipopt/lib/python3.12/site-packages/pyscipopt/../../.././././././libicudata.so.73: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0010001
BFD: warning: /mnt/irisgpfs/users/gmaudet/micromamba/envs/pyscipopt/lib/python3.12/site-packages/pyscipopt/../../.././././././libicudata.so.73: unsupported GNU_PROPERTY_TYPE (5) type: 0xc0010002
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
Core was generated by `python error_test.py'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0 0x00007f00a311ad1d in SCIPnodeFocus () from /mnt/irisgpfs/users/gmaudet/micromamba/envs/pyscipopt/lib/python3.12/site-packages/pyscipopt/../../../libscip.so.9.0
Missing separate debuginfos, use: yum debuginfo-install glibc-2.28-236.el8.7.x86_64 
```
This output informs us that the segmentation faults occurred in function `SCIPnodeFocus` of the `libscip.so.9.0` library.

You can get the full stack just by issuing the `bt` command to GDB:
```
(gdb) bt  
#0  0x00007f00a311ad1d in SCIPnodeFocus () from /mnt/irisgpfs/users/gmaudet/micromamba/envs/pyscipopt/lib/python3.12/site-packages/pyscipopt/../../../libscip.so.9.0                                                                          
#1  0x00007f00a30fccec in SCIPsolveCIP () from /mnt/irisgpfs/users/gmaudet/micromamba/envs/pyscipopt/lib/python3.12/site-packages/pyscipopt/../../../libscip.so.9.0                                                                           
#2  0x00007f00a30b73e8 in SCIPsolve () from /mnt/irisgpfs/users/gmaudet/micromamba/envs/pyscipopt/lib/python3.12/site-packages/pyscipopt/../../../libscip.so.9.0                                                                              
#3  0x00007f00a3d67f6d in __pyx_pw_9pyscipopt_4scip_5Model_377optimize () from /mnt/irisgpfs/users/gmaudet/micromamba/envs/pyscipopt/lib/python3.12/site-packages/pyscipopt/scip.cpython-312-x86_64-linux-gnu.so                              
#4  0x0000564e61a6adff in _PyObject_VectorcallTstate (kwnames=0x0, kwnames@entry=<error reading variable: dwarf2_find_location_expression: Corrupted DWARF expression.>, nargsf=9223372036854775809,                                          
   nargsf@entry=<error reading variable: dwarf2_find_location_expression: Corrupted DWARF expression.>, args=0x7f00b1fa7108, args@entry=<error reading variable: dwarf2_find_location_expression: Corrupted DWARF expression.>,              
   callable=0x7f009d3736b0, callable@entry=<error reading variable: dwarf2_find_location_expression: Corrupted DWARF expression.>, tstate=0x564e61eef3c8 <_PyRuntime+459656>,                                                                
   tstate@entry=<error reading variable: dwarf2_find_location_expression: Corrupted DWARF expression.>)  
   at /home/conda/feedstock_root/build_artifacts/python-split_1708115583107/_build_env/x86_64-conda-linux-gnu/sysroot/usr/include/bits/pycore_pyerrors.h:77                                                                                  
#5  PyObject_Vectorcall (callable=0x7f009d3736b0, args=0x7f00b1fa7108, nargsf=9223372036854775809, kwnames=0x0)  
   at /home/conda/feedstock_root/build_artifacts/python-split_1708115583107/_build_env/x86_64-conda-linux-gnu/sysroot/usr/include/bits/pycore_call.h:325                                                                                     
#6  0x0000564e61957a7e in _PyEval_EvalFrameDefault (tstate=<optimized out>, frame=0x7f00b1fa7090, throwflag=<optimized out>)                                                                                                                  
   at /home/conda/feedstock_root/build_artifacts/python-split_1708115583107/_build_env/x86_64-conda-linux-gnu/sysroot/usr/include/generated_cases.c.h:2706                                                                                   
#7  0x0000564e61b1ecae in PyEval_EvalCode (co=0x7f00b20dc4f0, globals=<optimized out>, locals=0x7f00a3ec5e00) at /usr/local/src/conda/python-3.12.2/Modules/abstract.c:578                                                                    
#8  0x0000564e61b4245a in run_eval_code_obj (tstate=0x564e61eef3c8 <_PyRuntime+459656>, co=0x7f00b20dc4f0, globals=0x7f00a3ec5e00, locals=0x7f00a3ec5e00) at Python/deepfreeze/stat.h:1722                                                    
#9  0x0000564e61b3dacb in run_mod (mod=<optimized out>, filename=0x7f00b20c2130, globals=0x7f00a3ec5e00, locals=0x7f00a3ec5e00, flags=0x7ffdc32421f0, arena=0x7f00b202bc50) at Python/deepfreeze/stat.h:1743                                  
#10 0x0000564e61b54230 in pyrun_file (fp=0x564e62173b20, filename=0x7f00b20c2130, start=<optimized out>, globals=0x7f00a3ec5e00, locals=0x7f00a3ec5e00, closeit=1, flags=0x7ffdc32421f0) at Python/deepfreeze/stat.h:1643                     
#11 0x0000564e61b53e0e in _PyRun_SimpleFileObject (fp=0x564e62173b20, filename=0x7f00b20c2130, closeit=1, flags=0x7ffdc32421f0) at Python/deepfreeze/stat.h:433                                                                               
#12 0x0000564e61b53684 in _PyRun_AnyFileObject (fp=0x564e62173b20, filename=0x7f00b20c2130, closeit=1, flags=0x7ffdc32421f0) at Python/deepfreeze/stat.h:78                                                                                   
#13 0x0000564e61b4d6b2 in pymain_run_file_obj (skip_source_first_line=0, filename=0x7f00b20c2130, program_name=0x7f00a3ec8c60) at /usr/local/src/conda/python-3.12.2/Programs/initconfig.c:360                                                
#14 pymain_run_file (config=0x564e61e91fa8 <_PyRuntime+77672>) at /usr/local/src/conda/python-3.12.2/Programs/initconfig.c:379                                                                                                                
#15 pymain_run_python (exitcode=0x7ffdc32421c4) at /usr/local/src/conda/python-3.12.2/Programs/initconfig.c:629  
#16 Py_RunMain () at /usr/local/src/conda/python-3.12.2/Programs/initconfig.c:709  
#17 0x0000564e61b09217 in Py_BytesMain (argc=<optimized out>, argv=<optimized out>) at /usr/local/src/conda/python-3.12.2/Programs/initconfig.c:763                                                                                           
#18 0x00007f00b11bad85 in __libc_start_main () from /lib64/libc.so.6  
#19 0x0000564e61b09095 in _start () at /home/conda/feedstock_root/build_artifacts/python-split_1708115583107/_build_env/x86_64-conda-linux-gnu/sysroot/usr/include/sys/pegen_errors.c:35620
```

## 2.3 Providing debug symbols for dynamically linked objects

Without debug information the core dump contains the call stack but only with addresses in memory, no function names. In fact, if the program links with functions in local shared libraries, you may have to install a package with the *debug symbols* of some libraries so that GDB can decode the function names corresponding to the memory addresses in the call stack.

# _Sources_

1. [ArchiWiki: Core dump](https://wiki.archlinux.org/title/Core_dump)
