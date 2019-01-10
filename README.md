# README #

Interface to run AFL on Java programs.

Kelinci means rabbit in Indonesian (the language spoken on Java).

***
### IMPORTANT ###

*This is the Kelinci-WCA version (please do 'git checkout kelinciwca'). The difference with standard Kelinci is that this version prioritizes costly paths. Time and memory consumption are measured on the Java side, sent back to the interface and then the special version of AFL also contained in this repo prioritizes paths that are more costly.*

*When you use this, make sure that all the components are those from this repo: instrumentor (re-instrument code!), interface.c and afl-fuzz. An extra flag for afl-fuzz allows to specify which resource to maximize: cost measured by instrumentation, time or memory (with `-c instrumentation`, `-c time` or `-c memory`, respectively), make sure you set it. The `instrumentation` cost model counts bytecode instructions using instrumentation, e.g. branches (jumps) or all instructions (depending on the actual instrumentation), and works best. The `time` cost model measures runtime but is too imprecise to direct fuzzing. Timing measurements can be useful later though, when studying results. The `memory` cost model measures heap usage and should in time be replaced by instrumentation as well. Additionally, KelinciWCA offers the possibility to use a user-defined cost model (with `-c userdefined`). Call `Kelinci.addCost(long)` directly in your code, where the parameter should be a positive long value. For slow running targets (like ours) it is recommended to use the -d flag with AFL, which skips deterministic fuzzing. Finally, make sure you use the -t flag for both afl-fuzz and the Kelinci server to set the time-out to something high enough, as time needed for processing input files will be increasing.*

*In addition to the standard AFL output, this version outputs a file path-costs.csv, which stores the costs for all inputs in the queue, crashes and hangs folders. Inputs that set a new record resource consumption are marked with `highscore` in the file name. If working correctly, one should notice resource usage increasing over time. Additionally, the current lowscore and highscore are listed on the last row, last 2 columns of the plot data file.*
***

This README assumes that AFL has been previously installed. For information on how to install and use AFL, please see <http://lcamtuf.coredump.cx/afl/>. Note: please replace the original afl-fuzz.c file with the one provided in "afl-2.51b-wca/" and make sure that AFL is built using this file.
Kelinci has been tested successfully with AFL versions 2.44 and newer. The README explains how to use the tool. For technical background, please see the CCS'17 paper in the 'docs' directory.

Several examples are provided in the 'examples' directory, where each comes with a README specifying the exact steps to take in order to repeat the experiment.

Kelinci has identified bugs related to JPEG parsing in OpenJDK (versions 6-9) and Apache Commons Imaging 1.0 RC7. Both are included in the provided examples. These are the bug reports:
- http://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8188756
- https://issues.apache.org/jira/browse/IMAGING-203

### Installation ###

The application has two components. First, there is a C application that acts as the target application for AFL.
It behaves the same as an application built with afl-gcc / afl-g++; AFL cannot tell the difference.
This C application is found in the subdirectory 'fuzzerside'. It sends the input files generated by AFL
to the JAVA side over a TCP connection. It then receives the results and forwards them to AFL in its
expected format. To build, run `make` in the 'fuzzerside' subdirectory.

The second component is on the JAVA side. It is found in the 'instrumentor' subdirectory.
This component instruments a target application with AFL style administration, plus a component to communicate
with the C side. When later executing the instrumented program, this sets up a TCP server and runs the target
application in a separate thread for each incoming request. It sends back an exit code (succes, timeout, crash
or queue full), plus the gathered path information. Any exception escaping main is considered a crash.
To build, run `gradle build` in the 'instrumentor' subdirectory.

### Usage ###

This describes how to run Kelinci on a target application. It assumes AFL and both Kelinci components have been built.

**1. Optional: build a driver**
AFL/Kelinci expects a program that takes a parameter specifying a file location. It randomly mutates files to fuzz this program. If this is not how your target application works, a driver will need to be built that parses an input file and simulates the normal interaction based on that. When building a driver, remember that the input files will be randomly mutated. The less structure and cohesion the program expects in the file, the more effective fuzzing will be. Even if the program does accept an input file, it could make sense to build a driver that accepts a different format in order to maximize the ratio of valid version invalid input files.

One more thing to take into account when building a driver is that the target program will be running in a VM where the main method will be invoked again and again. All runs must be independent and deterministic. If the program for instance stores information from an input in a database or a static memory location, be sure to reset it such that it cannot affect future runs.

**2. Instrument**
We'll assume that the target application and driver were built and the output directory is 'bin'. Our next step is to instrument the classes for use with Kelinci. The tool provides the edu.cmu.sv.kelinci.instrumentor.Instrumentor class for this. It takes an input directory after the `-i` flag (here 'bin') and an output directory after the `-o` flag (here 'bin-instrumented'). We'll need to make sure that the kelinci JAR is on the classpath, as well as all the dependencies of the target application. Additionally, one can define what kind of bytecode instructions should be instrumented by adding the parameter `-mode (jumps|labels)`.  `jumps` is the default value and instruments all byte code jump instructions (branches); `labels` instruments all labels in the bytecode, which effectively leads to an instrumentation of all instructions. Assuming the JARs that the target application depends on are in /path/to/libs/, the command to instrument looks like this:

```java -cp /path/to/kelinci/instrumentor/build/libs/kelinci.jar:/path/to/libs/* edu.cmu.sv.kelinci.instrumentor.Instrumentor -i bin -o bin-instrumented```

Note that there may be trouble if the project depends on different versions of libraries that the Kelinci Instrumentor also depends on. Currently, these are args4j version 2.32, ASM 5.2 and Apache Commons IO 2.4. In most cases, one can make this work by putting the 'classes' directory of the Kelinci build on the class path instead of the Fat JAR, then add a version of the library JARs to the classpath that both Kelinci and the target work with.

**3. Create example input**
We want to test that the instrumented Java application works now. To do this, create a directory for example input file(s):
```mkdir in_dir```

AFL will later use this directory to grab the input file(s) that it will mutate. It is therefor very important to have representative input files in there. Copy representative file(s) in there, or create them.

**4. Optional: test the Java application**
See if the instrumented Java application works with the provided / created input files:
```java -cp bin-instrumented:/path/to/libs/* <driver-classname> in_dir/<filename>```

**5. Start the Kelinci server**
We can now start the Kelinci server. We will simply edit the last command we executed, which ran the Java application. Kelinci expects the main class of the target application as the first parameter, so we can now just add the Kelinci main class before that one. We'll also need to replace the concrete file name by `@@`, which will be replaced by Kelinci with the actual path of the input file it wrote. Other parameters are ok and will be fixed across runs.

```java -cp bin-instrumented:/path/to/libs/* edu.cmu.sv.kelinci.Kelinci <driver-classname> @@```

Optionally, we can specify a port number (the default is 7007):
```java -cp bin-instrumented:/path/to/libs/* edu.cmu.sv.kelinci.Kelinci -port 6666 <driver-classname> @@```

**6. Optional: test the interface**
Before we start the fuzzer, let's make sure that the connection to the Java side is working as expected. The `interface.c` program has a mode for running outside of AFL as well, so we can test it as follows:
```/path/to/kelinci/fuzzerside/interface in_dir/<filename>```

If we created a list of servers in step 6, we can add it as follows:
```/path/to/kelinci/fuzzerside/interface -s servers.txt in_dir/<filename>```

Optionally, we can specify a server with the -s flag (e.g. -s 192.168.1.1 or "sv.cmu.edu", default is "localhost") and a port number with the -p flag (default is 7007).

**7. Start fuzzing!**
If everything works as expected, we can now start AFL! Like the Kelinci server side, AFL expects a binary that takes an input file as a parameter, specified by `@@`. In our case, this is always the `interface` binary. It also expects a directory with input files from which to start fuzzing, plus an output directory.

```/path/to/afl/afl-fuzz -i in_dir -o out_dir /path/to/kelinci/fuzzerside/interface [-s servers.txt] @@```

If everything works, the AFL interface will start up after a short while and you'll notice new paths being discovered. For additional monitoring, have a look at the output directory. The input files in the 'queue' subdirectory all trigger different program behaviors. There are also 'crashes' and 'hangs' subdirectories that contain inputs that resulted in a crash or a time-out. Please see the AFL website for more details: http://lcamtuf.coredump.cx/afl/

### Note on parallelization ###

The Java side is naturally parallelizable. Simply start an instance for every core you want to execute your Java runs on. This can be on the same machine (but different ports!) or on multiple machines.

For details on how to run AFL in parallel, please see the `parallel_fuzzing.txt` document that comes with it. You'll want to have as many afl-fuzz processes running as there are Java side Kelinci components, where each afl-fuzz process connects to a different Kelinci server. The Kelinci server to connect to can be specified using the `-s <server>` and `-p <port>` flags for `interface.c`.

### Developer ###

Rody Kersten (rodykersten@gmail.com)
