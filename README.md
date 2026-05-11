# Workshop Async Profiler and JDK Flight Recorder

Welcome to this workshop to discover the power of [async-profiler](https://github.com/async-profiler/async-profiler)
and/or [JDK Flight Recorder](https://dev.java/learn/jvm/jfr/)

## Requirements

### Run on your computer

- ⚠️ async-profiler only works for macOS or linux 
- ⚠️ JDK Flight Recorder works for all OS

Here are all the tools you need to have installed on your computer to run this workshop:

| Async-profiler                                                                      | JDK Flight Recorder                                                                   |
|-------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| [async-profiler](https://github.com/async-profiler/async-profiler/releases/)        | [Java Mission Control](https://adoptium.net/fr/jmc/)                                  |
| [Java 17+](https://adoptium.net/fr/)                                                | [Java 25+](https://adoptium.net/fr/) to benefit last features                         |
| [Docker Compose](https://docs.docker.com/compose/)                                  | [Docker Compose](https://docs.docker.com/compose/)                                    |
| [k6](https://k6.io/) (or [Docker](https://docs.docker.com/get-started/get-docker/)) | [k6](https://k6.io/) (or [Docker](https://docs.docker.com/get-started/get-docker/))   |
| [Java Mission Control](https://adoptium.net/fr/jmc/) (optional)                     |                                                                                       |


### Run on GitHub Codespaces

Create a GitHub Codespace directly from the repository

![codespace](./images/codespaces.png)

## Getting started

### Start the application

You are going to run a java application. This application has some dependencies that we will discover later.

In a terminal, please run this command to start the necessary dependencies:

```sh
docker compose up
```

Once it's done, let's start the application:

```sh
java -Xmx250m -Xms250m -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -XX:TieredStopAtLevel=1 -XX:FlightRecorderOptions:stackdepth=512 -jar workshop-async-profiler.jar
```

The application is listening on port 8080.

Make sure your application is correctly started by running:

```sh
curl http://localhost:8080/books
```

and

```sh
curl http://localhost:8090/new-books
```

---
**NOTE**

Some explanations about the java parameters:

 - `-Xmx250m` sets the maximum heap size of the JVM to 250 MB.
 - `-Xms250m` sets the initial (and minimum) heap size of the JVM to 250 MB: when you want to optimize GC work, it is a good practice to set `-Xms` with the same value as `-Xmx`
 - `-XX:+DebugNonSafepoints` this option ensures that the JVM records debug information at all points in the program (not just at safe points). Safe points are specific places in code where the JVM can pause execution for tasks like garbage collection, and this flag is useful for generating more accurate profiling information.
 - `-XX:+UnlockDiagnosticVMOptions` flag unlocks additional options for diagnosing faults or performance problems with the JVM.
 - `-XX:TieredStopAtLevel=1` disables intermediate compilation tiers (1, 2, 3). Setting this to 1 limits it to only the first level of compilation. We don't want our JVM to spend too much time on runtime optimization.
 - `-XX:FlightRecorderOptions:stackdepth=512` **provides non-truncated stack traces to JDK flight recorder**
---

> **For Async profiler** when the agent is not loaded at JVM startup (by using -agentpath option), it is highly recommended to use -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints JVM flags. Without those flags, the profiler will still work correctly, but results might be less accurate. For example, without -XX:+DebugNonSafepoints there is a high chance that simple inlined methods will not appear in the profile. When the agent is attached at runtime, the CompiledMethodLoad JVMTI event enables debug info, but only for methods compiled after attaching.
> [README](https://github.com/async-profiler/async-profiler/blob/master/docs/Troubleshooting.md#known-limitations)


### Warmup

Once the application has started correctly, let's inject some traffic into our application:

```sh
k6 run k6/warmup.js
```

If k6 is not installed, you can run this script using Docker. The base URL can be configured via an environment variable:

```sh
docker run --rm --add-host host.docker.internal:host-gateway -i grafana/k6 run -e BASE_URL=http://host.docker.internal:8080 - <k6/warmup.js
```

> **Note**: When using Docker on macOS or Windows, `host.docker.internal` is used to access the host from the container. On Linux, the `--add-host` option is required to add this host resolution.

The warmup script will:
- Run 10 virtual users (VUs)
- Each VU will execute 20 iterations
- Call both `/books` and `/new-books` endpoints
- Verify that 99% of requests complete within 1000 ms
- Maximum execution time is capped at 30 seconds

> [!important]
> ❓ Inspect the warmup file and the k6 report. Analyze the results.

## Profiling

### 🔥 Flamegraph

During our journey into profiling, we will generate flamegraphs to inspect our application. Here's a short introduction to flamegraphs:

A flamegraph is a visualization tool used to analyze performance bottlenecks in software, particularly for profiling CPU usage, memory, or execution time. It represents hierarchical data (like call stacks) in a compact, easy-to-read format, with the aim of showing where an application spends most of its time.

 - A flamegraph shows the function call hierarchy of a program, with each box representing a function or method in the call stack.
 - The x-axis represents the total time spent in a program, broken down by different functions. **The width of each box indicates how much time is spent on that particular function**.
 - The y-axis represents the call stack depth. Functions below them called functions higher up in the flamegraph.

Example:

![flamegraph example](./images/flamegraph_example.png)

How to read it:
 - `a()` calls `b()` and `h()`
 - `b()` calls `c()` and so on.
 - Here we can say `b()` takes more "resources" (CPU, memory, execution time) than `h()`.

Color Code:
 - 🔴 System (User native)
 - 🟢 Java
 - 🟡 C++

You can find more information about flamegraphs in the [Resources](#resources) section and [here](https://github.com/async-profiler/async-profiler/blob/master/docs/FlamegraphInterpretation.md).

### Finding the application PID

To use both async-profiler and JDK Flight Recorder, we need the PID (Process ID) of our Java application. Here are several ways to find it:

```sh
# Option 1: Using jps
export WORKSHOP_PID=$(jps -l | grep workshop-async-profiler.jar | cut -d' ' -f1)

# Option 2: Using pgrep (improved ps command)
export WORKSHOP_PID=$(pgrep -f workshop-async-profiler.jar)

# Option 3: If you know the port (8080 in our case)
export WORKSHOP_PID=$(lsof -t -i :8080)
```

The PID is stored in the `WORKSHOP_PID` environment variable  

### Wall-clock profiling

Wall-clock time (also called wall time) is the time it takes to run a block of code. 
The majority of applications deal with tiered components like a database, some HTTP or GRPC resources or a message broker (RabbiMQ, Apache Kafka, etc...), for example.
In those cases, the application spends most of its time on IO, waiting for those external components to respond.

#### Inject some traffic

During our profiling, we will inject some traffic using k6.

```sh
k6 run k6/main.js
```

If k6 is not installed, you can run this script using Docker. The base URL can be configured via an environment variable:


```sh
docker run --rm --add-host host.docker.internal:host-gateway -i grafana/k6 run -e BASE_URL=http://host.docker.internal:8080 - <k6/main.js
```

#### Our first Flamegraph

<details>
   <summary><b>With async-profiler</b></summary>
  `-e wall option` tells async-profiler to sample all threads equally every given period of time regardless of thread status: Running, Sleeping or Blocked.
  [README](https://github.com/async-profiler/async-profiler/blob/master/docs/ProfilingModes.md#wall-clock-profiling)

  Let's run the command during the traffic injection:
  ```sh
  cd /path/to/async-profiler-directory/bin
  ./asprof -e wall -f wall-1.html $WORKSHOP_PID
  ```
  async-profiler will sample during 60 seconds.

  Open the generated flamegraph in your favorite browser.
</details>

<details>
  <summary><b>With JDK Flight Recorder (JFR)</b></summary>
  Let's run the command during the traffic injection:
  
  ```sh
  # Capture CPU samples with JFR for 60s
  jcmd $WORKSHOP_PID JFR.start filename=wall.jfr duration=60s settings=cpu-sample.jfc
  ```

  the `jcmd` command with the option `JFR.Start` starts a JFR recording for a duration of 60 s. Record will be in the wall.jfr file.
  the record settings are defined in the cpu-sample.jfc: see [here](#more-about-jfc-sample-file-for-jfr) for more details about settings

  You can access the results via JDK Mission Control (JMC) :
   - Open the generated wall.jfr file in JMC
   - Open the flamegraph for _method profiling_ :
     - Difficult to interpret, isn't it!
  - Open the flamegraph for _threads_ and select `http-nio-exec*` threads :
    - Take note of the most used methods
  - Open the flamegraph with all events (click on _Event Browser_) :
    - are the same most used methods? 

  If you use Linux, you can run `jfr view cpu-time-hot-methods wall.jfr` (jfr is a JDK tool located at $JAVA_HOME/bin) which
  gives you directly the most used methods (experimental méthod only available for linux)

</details>


> [!important]
> ❓ Questions:
> - How many http requests have been done? 
> - What is the average duration and p9X?
> - What is the application doing?
> - where does the books from the endpoint `books` come from?
> - where does the books from the endpoint `new-books` come from?
> - what is taking more time?

<details>
  <summary><b>Measure effective time consumed to confirm your guess with JDK Flight Recorder</b></summary>
  Repeat the whole operation but this time enabling the `method-timing` parameter

  > Method timing records complete and exact statistics for method invocations
  > it has been introduced by [JEP 520 - Method timing and tracing](https://openjdk.org/jeps/520) in JDK 25 
  
  ```sh
  # Capture CPU samples with JFR for 60s
  jcmd $WORKSHOP_PID JFR.start filename=wall-timing.jfr duration=60s settings=cpu-sample.jfc method-timing=workshop.asyncprofiler.book.BookController
  ```

  - run `jfr view method-timing wall-timing.jfr`
</details>

<details>
   <summary><b>Let's see async-profiler per-thread mode</b></summary>
   Repeat the whole operation but this time using the option `-t`.

  > Wall-clock profiler is most useful in per-thread mode: -t.
  > [README](https://github.com/async-profiler/async-profiler/blob/master/docs/ProfilingModes.md#wall-clock-profiling)

   ```sh
   cd /path/to/async-profiler-directory/bin
   ./asprof -e wall -t -f wall-per-thread.html <pid>
   ```
</details>

> [!important]
> ❓ Count the number of Tomcat’s thread.

#### Add some latency

Let's inject some latency into the HTTP endpoint called by our application.

```sh
curl -s -XPOST -d '{"type" : "latency", "attributes" : {"latency" : 100}}' http://localhost:8474/proxies/wiremock/toxics
```

It adds 100 milliseconds latency.


<details>
   <summary><b>With async-profiler</b></summary>
   Let's repeat the operation of profiling and generate a flamegraph `wall-latency.html`.
   **To look for some methods in the flamegraph, you can use the shortcut CRTL+F to look for or use the magnifying glass 🔎.**
</details>


<details>
  <summary><b>With JDK Flight Recorder</b></summary>
  Let's repeat the operation of profiling and generate recording file `wall-latency.jfr`.
  ```sh
  jcmd $WORKSHOP_PID JFR.start filename=wall-latency.jfr duration=60s settings=cpu-sample.jfc
  ```
  
  Open the flamegraph in JMC for the threads view
  
  **To look for some methods in the flamegraph, you can type part of their fully qualified name in the input field at the top of the flamegraph.**
</details>



> [!important]
> ❓ In the flamegraph, look for the application's endpoints `/books` and `/new-books`.
> What is the main difference with the first flamegraph? Can you explain the differences?

Once you have finished your analysis, remove the latency using:

```sh
curl -XDELETE http://localhost:8474/proxies/wiremock/toxics/latency_downstream
```

### Memory Profiling

Add a function `authors` to the file `k6/main.js`. It should call the endpoint `/authors`.

```js
export function authors() {
    let res = http.get(`${BASE_URL}/authors`, { tags: { books: "authors" } });
    // Validate response status
    check(res, { "status was 200": (r) => r.status == 200 }, { books: "authors" });
}
```

Add k6 scenario configuration:

```json
"http_req_duration{books: \"authors\"}": ["p(99) < 1000"]

authors: {
   executor: 'per-vu-iterations',
   exec: 'authors',
   vus: 200,
   iterations: 500,
   maxDuration: '5m',
}
```

<details>
  <summary><b>With async profiler</b></summary>
  Let's profile the memory:
  
  ```sh
  cd /path/to/async-profiler-directory/bin
  ./asprof -e alloc -f memory.html <pid>
  ```

  Look at the flamegraph: memory allocations are now sampled
</details>


<details>
  <summary><b>With JDK Flight Recorder</b></summary>
  Let's profile the memory:

  ```sh
  jcmd $WORKSHOP_PID JFR.start filename=memory.jfr duration=60s settings=cpu-sample.jfc
  ```
  
  Check the "Automated analysis results" in JMC about Memory
  
  jfr can also give you a little clue : 
  
  ```shell
  jfr  view memory-leaks-by-site  memory.jfr
  ``` 

</details>

> [!important]
> ❓ Can you spot what is consuming more memory? Why?


#### Start profiling simultaneously with the application

Right now, we can't find which piece of code created the logging filter. We can assume it's a bean Spring that has been created at the application startup.
We need to profile code as soon as the JVM starts up.

<details>
  <summary><b>async-profiler as a Java agent</b></summary>

  > If you need to profile some code as soon as the JVM starts up, instead of using the asprof, it is possible to attach async-profiler as an agent on the command line.
  > [README](https://github.com/async-profiler/async-profiler/blob/master/docs/IntegratingAsyncProfiler.md)

  Stop your java application and launch it with this new parameter:
  
  ```sh
  java -agentpath:/path/to/async-profiler-directory/lib/libasyncProfiler.so=start,event="org.springframework.web.filter.AbstractRequestLoggingFilter.<init>" -Xmx250m -Xms250m -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -XX:TieredStopAtLevel=1 -jar workshop-async-profiler.jar
  ```
  
  The file `libasyncProfiler.so` can be found in the directory `lib` of the async-profiler.
  
  Once the application is started, you can run:
  
  ```sh
  cd /path/to/async-profiler-directory/bin
  ./asprof dump <pid>
  ```
</details>

<details>
  <summary><b>Starting JDK Flight Recorder with the JVM</b></summary>
  To start the JDK Flight Recorder with the JVM, we need to pass an option: `-XX:StartFlightRecording`.
  As we only need to which piece of code creates the `AbstractRequestLoggingFilter`, the recording will be configured only
  to capture this information :

  ```sh
  java -XX:FlightRecorderOptions:stackdepth=512 '-XX:StartFlightRecording:filename=init-trace.jfr,method-trace=org.springframework.web.filter.AbstractRequestLoggingFilter::<init>' -Xmx250m -Xms250m -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -XX:TieredStopAtLevel=1 -jar workshop-async-profiler.jar
  ```

  Once the application is started, dump the record :
  
  ```sh
  export WORKSHOP_PID=$(jps -l | grep workshop-async-profiler.jar|cut -d' ' -f1)
  jcmd $WORKSHOP_PID JFR.dump name=1
  ```

  See the result :
  
  ```shell
  jfr view --cell-height 10 --width 200  MethodTrace init-trace.jfr
  ```
</details>

Can you tell what is the instance of `AbstractRequestLoggingFilter`?

<details>
   <summary><b>Solution</b></summary>

   The memory allocation is due to the bean `CommonsRequestLoggingFilter` created in `WorkshopAsyncProfilerApplication`.

   This bean is configured with:

   ```java
   	    loggingFilter.setIncludePayload(true);
		loggingFilter.setMaxPayloadLength(5 * 1024 * 1024);
   ```

   For each HTTP request, an array of 5MB will be created.  

</details>


### CPU Profiling

> [!tip]
> Don't forget to restart the application without the async profiler java agent or the JDK Flight Recorder enabled 

A new endpoint has been developed and deployed. It computes the rating of an author based on all its written books.

Some say it's a heavy CPU consumer, let's find out.

Add this k6 configuration to the file `k6/main.js`. It should call the endpoint `/authorRating`.

```js
authorratings: {
   executor: 'per-vu-iterations',
   exec: 'authorRating',
   vus: 200,
   iterations: 500,
   maxDuration: '5m',
}


export function authorRating() {
    let authors= ["Madeline Miller","Erin Morgenstern","Tara Westover","Michelle Obama"]
    const randomIndex = Math.floor(Math.random() * authors.length);

    let res = http.get(`${BASE_URL}/author/${authors[randomIndex]}/rating`, { tags: { books: "author-rating" } });
    // Validate response status
    check(res, { "status was 200": (r) => r.status == 200 }, { books: "author-rating" });
}
```

<details>
  <summary><b>With async profiler</b></summary>
  You can run:
  
  ```sh
  cd /path/to/async-profiler-directory/bin
  ./asprof -e  cpu -f cpu.html <pid>
  ```
  
  you can face some issues profiling cpu event:
  
  ```sh
  [WARN] Kernel symbols are unavailable due to restrictions. Try
    sysctl kernel.perf_event_paranoid=1
    sysctl kernel.kptr_restrict=0
  [WARN] perf_event_open for TID 49766 failed: Permission denied
  ```
  
  There is a [dedicated section](https://github.com/async-profiler/async-profiler/blob/master/docs/Troubleshooting.md) to help you troubleshoot this issue.
  
  If changing the configuration is not possible, you may fall back to two options:
  
  `ctimer` profiling mode
  > It is similar to cpu mode but does not require perf_events support. As a drawback, there will be no kernel stack traces.
  
  ```sh
  cd /path/to/async-profiler-directory/bin
  ./asprof -e  ctimer -f cpu-ctimer.html <pid>
  ```
  
  `itimer` profiling mode.
  > Both cpu and itimer mode measure the CPU time spent by the running threads.
  > itimer mode is based on setitimer(ITIMER_PROF) syscall, which [ideally] generates a signal every given interval of the CPU time consumed by the process.
  > [Clarify samples count between -e cpu and -e itimer](https://github.com/async-profiler/async-profiler/issues/272)
  
  
  ```sh
  cd /path/to/async-profiler-directory/bin
  ./asprof -e  itimer -f cpu-itimer.html <pid>
  ```
</details>

<details>
  <summary><b>With JDK Flight Recorder</b></summary>
  Run again profiling :
  
  ```sh
  jcmd $WORKSHOP_PID JFR.start filename=cpu.jfr duration=60s settings=cpu-sample.jfc
  ```

  Have a look at "Automated analysis results" for methods profiling and open the flamegraph for method profiling for in JMC
  
</details>

> [!important]
> ❓ Generate all the flamgraphs and analyze the results.

<details>
  <summary><b>Confirm that CPU consumption impact method execution time with JDK Flight Recorder</b></summary>
  Run again profiling :

  ```sh
  jcmd $WORKSHOP_PID JFR.start filename=cpu-timing.jfr duration=60s settings=cpu-sample.jfc method-timing=java.util.concurrent.atomic.AtomicLong\;workshop.asyncprofiler.book.BookController
  ```

  Then :
  
  ```shell
  jfr view method-timing cpu-timing.jfr
  ```
</details>

### Multiple Events (optional)

It's possible to profile multiple events at the same time. For example, you can profile CPU, allocations, and locks at the same time. You may choose any other execution event instead of CPU, like wall-clock.

**The only output format that supports multiple events is JFR**.

Let's profile the application:

```sh
cd /path/to/async-profiler-directory/bin
./asprof -e wall,alloc,lock -f profile.jfr <pid>
````

Then, you can open the JFR file using [Java Mission Control](https://adoptium.net/fr/jmc/)

![JMC Memory](./images/jmc_memory.png)

## Resources

Here's a list of resources that helped me build this workshop.

- [async-profiler](https://github.com/async-profiler/async-profiler)
- [jvmperf](https://jvmperf.net/)
- [Coloring Flame Graphs: Code Hues](https://www.brendangregg.com/blog/2017-07-30/coloring-flamegraphs-code-type.html) by Brendan Gregg
- [A Guide to async-profiler](https://www.baeldung.com/java-async-profiler) by Anshul Bansal and Eric Martin
- [USENIX ATC '17: Visualizing Performance with Flame Graphs](https://www.youtube.com/watch?v=D53T1Ejig1Q) by Brendan Gregg
- [Taming performance issues into the wild: a practical guide to JVM profiling](https://www.youtube.com/watch?v=Cw4nN5L-2vU) by Francesco Nigro, Mario Fusco
- [[Java][Profiling] Async-profiler - manual by use cases](https://krzysztofslusarski.github.io/2022/12/12/async-manual.html) by Krzysztof Ślusarski
- [[Java][Profiling][Memory leak] Finding heap memory leaks with Async-profiler](https://krzysztofslusarski.github.io/2022/11/27/async-live.html) by Krzysztof Ślusarski
- [Java Safepoint and Async Profiling](https://seethawenner.medium.com/java-safepoint-and-async-profiling-cdce0818cd29) by Seetha Wenner
- 🇫🇷 [Traquer une fuite mémoire :cas d’étude avec Hibernate 5, ne tombez pas dans le IN !](https://www.sfeir.dev/back/traquer-une-fuite-memoire-cas-detude-avec-hibernate-5-ne-tombez-pas-dans-le-in/)by Ling-Chun SO
- 🇫🇷 [Sous le capot d'une application JVM - Java Flight Recorder / Java Mission Control](https://www.youtube.com/watch?v=wa_EtTUx-z0) by Guillaume Darmont
- 🇫🇷 [Performance et diagnostic - Méthodologie et outils](https://speakerdeck.com/vladislavpernin/performance-et-diagnostic-methodologie-et-outils) by Vladislav Pernin

## More about JFC sample file for JFR

JDK FLight Recorder works by recording many events emitted by the JVM and the application. Many events like creating a new instance of a class,
calling a method, starting a GC, or reading bytes from a file are predefined in JDK. A settings file sets events enabled for each recording.
Two setting files exist in the JDK :
1. `lib/jfr/default.jfc` : for current profiling: record fewer events with lower sample rate but with a lower overhead which is acceptable for production use: this is the enabled default
2. `lib/jfr/profile.jfc` : it records more events at a higher rate for diagnosis profiling with higher overhead

One can write its own setting file from `default.jfc` or `profile.jfc` for a particular use: this is what we did for this workshop: 
we derived the `cpu-sample.jfc` from `profile.jfc` :
- Events disabled for security (avoid potential sensitive data or configuration leaks) :
  - jdk.InitialSystemProperty
  - jdk.SystemProcess
  - jdk.InitialEnvironmentVariable
  - jdk.InitialSecurityProperty
- Event enabled for performance sample purpose: jdk.CPUTimeSample with a throttle of 2 ms

Recording can also be configured with command line: that is what we did when we run commands such as `jcmd $WORKSHOP_PID JFR.start filename=wall-timing.jfr duration=60s settings=cpu-sample.jfc method-timing=workshop.asyncprofiler.book.BookController`
which also enable event jdk.MethodTiming for timing all calls to the methods of the class BookController: this kind of recording settings,
in particular if you time many methods' execution, may have a significant additional overhead.