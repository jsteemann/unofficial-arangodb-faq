# The Unofficial ArangoDB FAQs

This is an opinionated collection of unofficial FAQs, recipes and tips for the 
[ArangoDB multi-model database](https://www.arangodb.com/).

* [Building ArangoDB](#building-arangodb)
  * [How to compile ArangoDB from source?](#how-to-compile-arangodb-from-source)
  * [How to fix a broken build directory?](#how-to-fix-a-broken-build-directory)
  * [What are the most common build options?](#what-are-the-most-common-build-options)
  * [I compiled with ASan, TSan etc. but it doesn't work!?](#i-compiled-with-asan-tsan-etc-but-it-doesnt-work)
  * [How to work with multiple versions of ArangoDB?](#how-to-work-with-multiple-versions-of-arangodb)
* [Inspection](#inspection)
  * [How to find out which version of ArangoDB I am running?](#how-to-find-out-which-version-of-arangodb-i-am-running)
  * [What are the available startup parameters of ArangoDB](#what-are-the-available-startup-parameters-of-arangodb)
  * [What are useful logging options for debugging and developing](#what-are-useful-logging-options-for-debugging-and-developing)

I will add items to this list as I personally see fit.

I am a developer using Linux, so I will most likely curate items here that have more 
relevance for people that like using the command-line rather than fancy user interfaces.

## Building ArangoDB

### How to compile ArangoDB from source?

As a prerequisite, install CMake and a C++ compiler of your choice (clang++ or g++, recent versions 
will do), as well as the OpenSSL library's development files.

Get the ArangoDB source code by cloning the main ArangoDB Github repository.
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

In this case, rather use the .tar.gz packages provided on the [ArangoDB download page](https://download.arangodb.com/),
use the ArangoDB starter, compile ArangoDB yourself, or use some other custom deploy mechanism.

Whenever running multiple ArangoDB processes in parallel on the same host, please make sure they
are configured to bind to different ports and use different database directories. Otherwise the 
processes will refuse to start.


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

Note that these options can only be set at server start.
