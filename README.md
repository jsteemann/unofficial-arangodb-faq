# The Unofficial ArangoDB FAQs 🥑📖🔥

This is an opinionated collection of unofficial FAQs, recipes and tips for the 
[ArangoDB multi-model database](https://www.arangodb.com/).

* [Building ArangoDB from source](#building-arangodb-from-source)
  * [How to compile ArangoDB from source?](#how-to-compile-arangodb-from-source)
  * [How to fix a broken build directory?](#how-to-fix-a-broken-build-directory)
  * [What are the most common build options?](#what-are-the-most-common-build-options)
  * [I compiled with ASan, TSan etc. but it doesn't work!?](#i-compiled-with-asan-tsan-etc-but-it-doesnt-work)
  * [How to work with multiple versions of ArangoDB?](#how-to-work-with-multiple-versions-of-arangodb)
* [Testing and debugging](#testing-and-debugging)
  * [How to run ArangoDB tests?](#how-to-run-the-arangodb-tests)
  * [How to run a specific test suite until it fails?](#how-to-run-a-specific-test-suite-until-it-fails)
  * [How to tap into the HTTP communication between ArangoDB servers?](#how-to-tap-into-the-http-communication-between-arangodb-servers)
* [Inspection](#inspection)
  * [How to find out which version of ArangoDB I am running?](#how-to-find-out-which-version-of-arangodb-i-am-running)
  * [What are the available startup parameters of ArangoDB](#what-are-the-available-startup-parameters-of-arangodb)
  * [What are useful logging options for debugging and developing](#what-are-useful-logging-options-for-debugging-and-developing)
* [OS configuration](#os-configuration)
  * [What are the most important OS settings for ArangoDB?](#what-are-the-most-important-os-settings-for-arangodb)
* [ArangoDB configuration](#arangodb-configuration)
  * [How to configure the number of threads?](#how-to-configure-the-number-of-threads)
  * [How to configure the number of V8 contexts?](#how-to-configure-the-number-of-v8-contexts)
  * [How to turn off statistics gathering?](#how-to-turn-off-statistics-gathering)
  * [How to turn off Foxx queues?](#how-to-turn-off-foxx-queues)

I will add items to this list as I personally see fit.

I am a developer using Linux, so I will most likely curate items here that have more 
relevance for people that like using the command-line rather than fancy user interfaces.

This is a personal repository, and everything published here are my personal views and recommendations.
ArangoDB Inc. may have different official recommendations. 
Whenever in doubt, I suggest following the company's official recommendations!


## Building ArangoDB from source

### How to compile ArangoDB from source?

As a prerequisite, install CMake and a C++ compiler of your choice (clang++ or g++, recent versions 
will do), as well as python2.7 and the OpenSSL library's development files:
```bash
sudo apt install cmake g++ python2.7 libssl-dev
```

Then get the ArangoDB source code by cloning the main ArangoDB Github repository.
The default branch is `devel`. If you want to use a different branch (e.g. `3.4`) just clone that.
The repository has quite some history and with full history it will be more than one GB to download.

Clone branch `3.4` into (new) directory `arangodb-3.4`:
```bash
git clone -b 3.4 https://github.com/arangodb/arangodb arangodb-3.4
cd arangodb-3.4
```

Create an initial `build` subdirectory and run `cmake` to configure the build:
```bash
mkdir build
(cd build && cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo ..)
```

If the cmake command reported any errors, they need to be fixed first before you can go on.
Once cmake is happy, it's time to start the actual build:
```bash
(cd build && make -j16)
```
This invokes up to 16 parallel processes for building. Adjust to the number of cores on your system
to speed up the build. Don't use a higher number than there are cores available.

The initial build may take a long time as it will need to build not only ArangoDB but also the
required libraries such as curl, ICU, V8, Boost and RocksDB.

Further builds should be quicker, as the build is incremental.

Once the build is ready, you will have all the binaries readily available in the `build`
subdirectory.

There is no need to run `make install`.

You can start arangod, arangosh, arangoimport etc. directly from the directory you are in,
e.g. `build/bin/arangod -c etc/relative/arangod.conf /path/to/database-directory`.

I recommend to not use `cd` to move back and forth between the checkout directory and the
`build` subdirectory. That will only waste time. Use a subshell to move into the `build` subdirectory
only temporarily, as in all the commands above. With that approach you will be able to use your
shell's command-line history and can easily navigate between the last few commands in your history.

### How to fix a broken build directory?

Builds are incremental, so starting another build will only compile and link the parts that
have changed. This should normally workly nicely. However, in rare occasions the build system
fails to detect the dependencies properly and does not build all the required parts. In that case
you will likely see cryptic linker errors and such.

In this case, selectively removing parts from the `build` subdirectory may help, e.g.
```bash
rm -rf build/arangod buil/bin/arangod
```
The above will make the next build rebuild everything for the `arangod` executable from scratch.
If errors persist, try wiping your entire `build` subdirectory and restart with the `cmake`
command from scratch.

### What are the most common build options?

When invoking `cmake` to configure an ArangoDB build, there are a lot of build options to set.
Most of them are irrelevant to normal users, but for developers there are some interesting
tweaks:

* `CMAKE_BUILD_TYPE`: `Debug`, `Release` or `RelWithDebInfo`. `Debug` builds normally build faster,
  play nice with a debugger and behave developer-friendly when looking at their coredumps. But they
  are slow. So don't use them for performance-testing and such. `Release` builds have optimizations
  turned on, so they take longer to build but run a lot faster. However, they lack debug information
  so as a developer you don't want them. `RelWithDebInfo` is a compromise: the binaries are
  fast but have debug symbols, so debugging is still possible. However, when debugging these binaries
  you will often see variables being "optimized out" in the debugger or in coredumps.

  Rule of thumb:
  - when developing ArangoDB features, use `Debug` or `RelWithDebInfo`
  - when using ArangoDB on a production site to debug issues, use `RelWithDebInfo`
  - when using ArangoDB on a production site for normal operations, use `Release`

  The default is `Release`.

* `USE_MAINTAINER_MODE`: `On` or `Off`. Turning on the maintainer mode will compile assertions into
  the binary, which are useful when developing new features or running the tests. Assertions slow 
  down the executable a bit, but in most cases the slowdown will be a few percents only.

  When turning on maintainer mode, you will additionally need to install Bison and Flex to generate
  code from Bison grammar files and Flex source files. 

  Rule of thumb:
  - when developing ArangoDB features or trying to debug some issues, use `On`
  - when using ArangoDB on a production site for normal operations, use `Off`

  The default is `Off`.

* `USE_FAILURE_TESTS`: `On` or `Off`. Failure tests compile in certain failure points that can be
  triggered from within test code. This is only meaningful when writing or running tests, or when
  developing ArangoDB features. Enabling failure tests can slow some hot code paths down considerably,
  and thus should not be done when conducting any performance tests.
  
  Rule of thumb:
  - when developing ArangoDB features or running the tests, use `On`
  - when using ArangoDB on a production site for normal operations, use `Off`
  
  The default is `Off`.

* `USE_JEMALLOC`: `On` or `Off`. Toggles usage of the [jemalloc](https://github.com/jemalloc/jemalloc) 
  memory allocator in ArangoDB. jemalloc is a thread-caching memory allocator and may speed up
  several workloads by a great deal. It may have different memory usage characteristics than
  your system's default memory allocator, so it may be worth trying to run ArangoDB without it
  and compare memory usage.

  jemalloc needs to be turned off when running ArangoDB with Valgrind, or with compiled-in
  ASan or TSan support (see below).

  The default is `On` on Linux and Mac OS.

* `USE_OPTIMIZE_FOR_ARCHITECTURE`: `On` or `Off`: Whether or not to optimize the executables for 
  the build host platforms. Use this if you want to use the binaries on the same machine/architecture
  that you build them on. The produced binaries may then not be portable, as they may contain
  architecture-specific optimizations. This is normally what you want if you compile yourself,
  except if you want to build portable packages for distribution. In this case, turn the option off.

  The default is `On`.

### I compiled with ASan, TSan etc. but it doesn't work!?

ArangoDB by default uses the jemalloc memory allocator, which overrides the same memory management
hooks as the Address Sanitizer (ASan) and Thread Sanitizer (TSan). If you want to use any of these,
make sure you configure the build to not use jemalloc as well, by setting the `USE_JEMALLOC` cmake
option to `Off`:

```bash
(cd build && cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo -DUSE_JEMALLOC=Off ..)
(cd build && make -j16)
```

### How to work with multiple versions of ArangoDB?

Having multiple versions of ArangoDB on the same host is not a problem in general, but requires
some caution.

In case multiple different versions shall be used on the same machine, don't use commands such as
`make install` from an ArangoDB build directory, as one version will happily overwrite the data
of another. Instead, use local build directories and start the different versions from inside there,
without clobbering the shared system directories.

Using one of the official ArangoDB release packages will always install the ArangoDB version
contained in the package into the shared system directories and also start that particular 
version as a service binding to port 8529 and using the database directory `/var/lib/arangodb3`.
This is normally not what is desired when multiple instances should be run.

In this case, rather use the .tar.gz packages provided on the [ArangoDB download page](https://www.arangodb.com/download-major/),
use the ArangoDB starter, compile ArangoDB yourself, or use some other custom deploy mechanism.

Whenever running multiple ArangoDB processes in parallel on the same host, please make sure they
are configured to bind to different ports and use different database directories. Otherwise the 
processes will refuse to start.

## Testing and debugging

### How to run ArangoDB tests?

As a prerequisite for running the tests, please make sure that you have built ArangoDB in maintainer 
mode. Additionally it is useful to enable failure tests when building, because that will produce
more tests that can be executed.

To run ArangoDB's bundled tests, run `scripts/unittest all` on the command-line when inside the
source checkout directory.

This will start single-server tests for the default storage engine, i.e. RocksDB from version 3.4
onwards, and MMFiles before. To run tests for a different storage engine, use the `--storageEngine`
option. To run tests for the cluster, use the `--cluster` option.

Here are a few copy&paste snippets for starting tests:
```bash
scripts/unittest all --storageEngine mmfiles --cluster false
scripts/unittest all --storageEngine mmfiles --cluster true
scripts/unittest all --storageEngine rocksdb --cluster false
scripts/unittest all --storageEngine rocksdb --cluster true
```

In contrast to its name, the target `all` does not run all tests, but only the most common ones.
To get an idea of what tests are included in `all` and which aren't, you can invoke `scripts/unittest`
without any further parameters. This will show a list of available test suites and whether or not
each is included in `all`.

Some important tests that are missing in `all` are the recovery and replication tests. To run them
too, use something like
```bash
scripts/unittest all recovery shell_replication http_replication replication_sync replication_static replication_ongoing replication_random replication_aql replication_fuzz --storageEngine mmfiles
scripts/unittest all recovery shell_replication http_replication replication_sync replication_static replication_ongoing replication_random replication_aql replication_fuzz --storageEngine rocksdb
```
Note: these tests only exist in single-server mode.

### How to run a specific test suite until it fails?

If a test fails sporadically or seems non-deterministic, a good way to force failure is to run
the test in an endless loop until it fails, e.g. by using the following bash script:

```bash
while true; do 
  scripts/unittest shell_server --test tests/js/server/shell/shell-skiplist-index.js
  if [ "$?" -ne "0" ]; then 
    break; 
  fi; 
done

Add some background load to the mix, e.g. by compiling ArangoDB in another directory, and wait
until the failure occurs again.

### How to create a backtrace of all threads of a running ArangoDB process?

As a prerequisite, you will need to install `gdb`.

You then need to find out the process id (pid) of the arangod process of interest. If there is 
only arangod process running on the server, you can use `pidof` if installed. Otherwise, just
insert the process id into the following command:

```bash
echo "thread apply all bt full" | sudo gdb -p `pidof arangod` > backtrace.out
```

After that the backtrace data will be stored in file `backtrace.out`.

### How to tap into the HTTP communication between ArangoDB servers?

Though deprecated, `ngrep` will do a good job. If servers are using ports 8530, 8629 and 8630,
the following command will sniff the communication done via these ports and print it on screen:

```bash
sudo ngrep -W byline -d lo port 8530 or port 8629 or port 8630
```

## Inspection

### How to find out which version of ArangoDB I am running?

If you have an `arangod` executable available, the easiest way to find its version number is to 
invoke it with the `--version` command-line option:

```bash
arangod --version
```
This will print the version number first, followed by a lot of other build details. Among other
things, this will tell you whether you are using a debug or release build, the date and time it
was built and a few other things that may be relevant.

If you don't have an `arangod` executable available but a few server process to inspect, you can
use `curl` and send a request to the `/_api/version` endpoint. Please note that you will need
authentication details for the server processes in order to connect.

Example:
```bash
curl "http://127.0.0.1:8529/_api/version" --dump - --user "username:password"
```
Adjust host IP address and port as well as username and password as needed. When successful, this
should also provide the version number of the executable.
To get more details, append the `details` URL parameter as follows:
```bash
curl "http://127.0.0.1:8529/_api/version?details=true" --dump - --user "username:password"
```

Apart from that, the login screen of the ArangoDB web interface will also show the version 
number.

### What are the available startup parameters of ArangoDB?

When in doubt about any of the numerous startup parameters of `arangod` or any of the bundled
client tools, the executables provide a way to list all their known parameters via the `--help`
option, e.g.
```bash
arangod --help
arangosh --help
```
Actually `--help` will list only the most common startup parameters, but will omit a few 
esoteric ones.
To get the full list of parameters, use `--help-.`. This will print *all* parameters, so it is
recommended to pipe that output to a pager for more comfort:
```bash
arangod --help-. | less
```
There is also the alternative of listing only the options of certain parameter sections, by
specifying `--help-` followed by the section name, e.g.
```bash
arangosh --help-log
```
The parameter help output will also contain the default values for the parameters, which can
be quite handy when in doubt.

### What are useful logging options for debugging and developing?

When developing or debugging ArangoDB features, it may be useful to increase the log level of
dedicated log topics.
By default, most log topics are set to log only messages of level `INFO` or higher (that is
`INFO`, `WARN`, `ERR` and `FATAL`). To get more details, you can try setting the log level for
the topics of interest to `DEBUG` or `TRACE`. This can be done using the `--log.level` startup
option, which can also be specified multiple times:

```bash
arangod --log.level communication=trace --log.level requests=debug --log.level queries=debug ...
```

Important log topics for common problems are:

* `queries`: used for AQL queries
* `replication`: replication progress (mostly on the follower)
* `config`: config file and option processing
* `startup`: startup sequence and order
* `communication`: request/response data, VST
* `requests`: request/response data, request handling
* `heartbeat`: heartbeat status messages

Adjust these as needed.

To find out at runtime which log topics exist, you can issue a curl request to the server:
```bash
curl "http://127.0.0.1:8529/_admin/log/level" --dump - --user "username:password"
```
Using curl it is also possible to adjust the log levels for dedicated topics for a running
server without restarting it:

```bash
curl -X PUT "http://127.0.0.1:8529/_admin/log/level" --dump - --user "username:password" -d '{"requests":"TRACE","communication":"TRACE"}'
```

It may also be useful to turn on additional details for the logging. The following options 
are most useful:

* `--log.line-number true`: show filename and line number of the code part producing the log
  message
* `--log.thread true`: show thread identifiers in log messages in addition to the process id. 
  this is useful when debugging issues that involve multiple threads
* `--log.force-direct true`: log messages are by default buffered, and written to the terminal
  or the logfile asynchronously. this is normally not a problem, but in combination with other
  test output this may cause log messages to appear out of line. in order to emit log messages
  are soon as they are created, set this option
* `--log.role true`: show server role using a single letter (e.g. *C* for coordinator, *A* for
  agent etc.). use this when looking at the logs of multiple instances in a cluster to tell
  the instances apart more easily by their roles

Note that these options can only be set at server start but not changed at runtime.


## OS configuration

### What are the most important OS settings for ArangoDB?

The most important settings for ArangoDB are

* the number of open file descriptors per process: it is essential for normal operations that
  this value is high enough, i.e. at least 1024 for RocksDB, and even higher for the MMFiles engine.
  The higher the better. The setting can be adjusted for the current session by using `ulimit -n` 
  or permanently by setting the `nofile` value for the correct user/group in `/etc/security/limits.conf`

* memory configuration: by default, the Linux kernel will overcommit memory based on some heuristics.
  When the kernel cannot provide more memory, it may kill memory-hogging processes by invoking its
  OOM killer. On dedicated database servers, the processes that use most memory are likely `arangod`
  processes, so the kernel will kill these first to make memory available. To avoid this situation,
  the kernel setting `overcommit_memory` should be set to a value of `2`. This will make the kernel
  deny memory allocation requests in case no more memory is available, so that applications can handle
  the situation more gracefully.

  An `overcommit_memory` value of `2` will make the kernel hand out as much memory as there is swap
  space available, plus a configurable fraction of physical RAM. This fraction can be adjusted via
  the `overcommit_ratio` kernel parameter. Please be aware that the default value is only `50`, meaning
  only 50% of physical RAM will be handed out as committed memory. So it makes sense to increase that
  ratio a lot, to something around 90.

To view the current memory configuration, use 

```bash
cat /proc/sys/vm/overcommit_memory 
cat /proc/sys/vm/overcommit_ratio 
```

To temporarily adjust these values, use something like:
```bash
sudo bash -c "echo 2 > /proc/sys/vm/overcommit_memory"
sudo bash -c "echo 90 > /proc/sys/vm/overcommit_ratio"
```

To make these changes durable, persist them in `/etc/sysctl.conf`.

Another important memory-related setting is `max_map_count`, which controls the maximum number of
memory mappings a process is allowed to have. The kernel default value is something around 64K, which
may be too low. Raising this value to 1M or even higher is thus strongly recommended for running
ArangoDB.

Again, setting the value once can be done by patching the file in the `/proc` filesystem directly:

```bash
sudo bash -c "echo 10000000 > /proc/sys/vm/max_map_count"
```

The change can be made durable by persisting the change in `/etc/sysctl.conf`.

If you want to enable coredumps, it is advisable to use either `ulimit -c unlimited` for the current
session, or persist the `core` setting in `/etc/security/limits.conf` for the correct user/group.


## ArangoDB configuration

ArangoDB's default settings are reasonable for most use cases. However, a few specific options
may be fine-tuned.

### How to configure the number of threads?

From ArangoDB 3.4 on, the number of threads of an arangod process can be effectively limited by
setting the `--server.maximal-threads` value to the desired number of threads. Earlier ArangoDB
versions also provided this setting, but the effective minimum value for the number of threads
was always 64, even if a lower value was used in the configuration. ArangoDB 3.4 will honor the
configured maximum number of threads for its pool of scheduler threads.

### How to configure the number of V8 contexts?

The V8 JavaScript enigne is used inside ArangoDB for executing user-defined Foxx code, AQL
user-defined functions and a few other places. This is the situation as in ArangoDB 3.4. Earlier
releases used V8 for several other tasks, for example executing plan changes in the cluster,
some AQL functons, and some of the standard REST APIs. This means releases before ArangoDB 3.4
may need more V8 contexts around than 3.4 and higher, in which the usage of V8 is mostly reduced
to running user-defined code.

Agency nodes and cluster database servers do not need to execute user-defined code, so they are
started with the V8 engine turned off starting with ArangoDB 3.4 on. There is thus no need to set
the number of V8 contexts for agency nodes and cluster database servers in 3.4 and higher. For
single-server instances and cluster coordinator servers, V8 is still needed to run user-defined
code, serve ArangoDB's web interface and a few other things. Setting the number of V8 contexts for 
these types of servers is normally not required in 3.4 and higher, as the number of contexts will
dynamically float based on the number of V8 contexts requested. 

In 3.4, only in case user-defined AQL functions, Foxx or JavaScript transactions are used a lot, 
it may make sense to adjust the number of V8 contexts by using the following settings. In earlier
versions of ArangoDB, it may make sense to adjust the number of contexts in more cases, as V8 is
used for more types of operations there.

The `--javascript.v8-contexts-minimum` value determines the minimum number of contexts that are
kept around. The setting `--javascript.v8-contexts` determines the maximum number of V8 contexts.
The actual number of contexts will float between these two settings as needed. Unused contexts
will be disposed after a while of inactivity until the number of contexts reaches the configured
lower bound.

Please note that each V8 context requires a significant amount of memory, so the number of contexts
should not be increased needlessly.

### How to turn off statistics gathering?

By default, arangod servers will periodically gather some statistics and store them in the 
database. The statistics can be retrieved via a REST API later or be viewed via the web interface.
If these statistics are of no interest, they can be turned off at server start. ArangoDB will 
still be functional, yet it will not gather or display any statistics.

The configuration option for turning off the statistics is `--server.statistics true`.

### How to turn off Foxx queues?

Foxx queues are a mechanism for dispatching one-off or recurring user-defined jobs in ArangoDB. 
By default, arangod servers will periodically check for jobs in Foxx queues every second. This 
will put a small load on the database which can be avoided if Foxx queues are not used in a 
particular setup. With the queues turned off, ArangoDB will still be functional, only that it
will not pick up any jobs dispatched via Foxx queues.

Specifying `--foxx.queues false` when starting the server will turn the queues off.
