# Introduction

The aim of this repo is to measure some features of an Micronaut application and its Docker image. 
Instead of creating a new Micronaut demo from scratch, it is based on [Micronaut migration](https://github.com/graemerocher/micronaut-petclinic) of [Spring PetClinic](https://github.com/spring-projects/spring-petclinic).

#  Size

## Micronaut artifact's size

Build PetClinic application:

```
    git clone https://github.com/wearearima/micronaut-docker-study.git
    cd micronaut-docker-study
    ./mvnw clean package
```

Measure the jar files:

```
    ls -lh target/*.jar*
```

Result:

```
-rw-r--r--  1 ugaitz  staff    36M Sep  4 16:41 target/micronaut-petclinic-0.1.0.BUILD-SNAPSHOT.jar
-rw-r--r--  1 ugaitz  staff   605K Sep  4 16:40 target/original-micronaut-petclinic-0.1.0.BUILD-SNAPSHOT.jar
```

The file named `micronaut-petclinic-0.1.0.BUILD-SNAPSHOT` is the resulting fat jar because it includes
PetClinic's code and its dependencies. The other file, suffixed `.original`, is just PetClinic's code
without its dependencies. The result is that our code size is `605K` and the dependencies' `36M`. 


## Docker image's size

The original Micronaut PetClinic application already has a Dockerfile to build and run project in GraalVM, 
for our measurements we have created another Dockerfile.
Build PetClinic's Docker image:

```
    docker-compose -f docker-compose-runjava.yml build
```

Measure the image size:

```
    docker image ls | grep micronaut-petclinic-java
```

Result:

```
micronaut-petclinic-java        latest              3c0f411ba9c1        15 hours ago        143MB
```

We can see that size of the artifact has increased from `36MB` to `143MB`. This is mainly because the 
Docker image includes the JDK and Linux images. Run this command to check it:

```
    docker image history micronaut-petclinic-java
```

The result shows the different layers added to the Docker image:

```
➜  micronaut-docker-study git:(master) ✗ docker image history micronaut-petclinic-java
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
3c0f411ba9c1        15 hours ago        /bin/sh -c #(nop)  CMD ["/bin/sh" "-c" "/usr…   0B                  
bf33527b993f        15 hours ago        /bin/sh -c #(nop)  EXPOSE 8080                  0B                  
df1e49f75aa4        15 hours ago        /bin/sh -c #(nop) COPY file:75746e4928644a0a…   37.8MB              
8bced513ccd7        16 hours ago        /bin/sh -c #(nop)  ARG JAR_FILE                 0B                  
f2ff0b12ec0f        16 hours ago        /bin/sh -c #(nop)  VOLUME [/tmp]                0B                  
a3562aa0b991        3 months ago        /bin/sh -c set -x  && apk add --no-cache   o…   99.3MB              
<missing>           3 months ago        /bin/sh -c #(nop)  ENV JAVA_ALPINE_VERSION=8…   0B                  
<missing>           3 months ago        /bin/sh -c #(nop)  ENV JAVA_VERSION=8u212       0B                  
<missing>           3 months ago        /bin/sh -c #(nop)  ENV PATH=/usr/local/sbin:…   0B                  
<missing>           3 months ago        /bin/sh -c #(nop)  ENV JAVA_HOME=/usr/lib/jv…   0B                  
<missing>           3 months ago        /bin/sh -c {   echo '#!/bin/sh';   echo 'set…   87B                 
<missing>           3 months ago        /bin/sh -c #(nop)  ENV LANG=C.UTF-8             0B                  
<missing>           3 months ago        /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B                  
<missing>           3 months ago        /bin/sh -c #(nop) ADD file:a86aea1f3a7d68f6a…   5.53MB 
```

The `5.53MB` image is the Alpine Linux image and the `99.3MB` image is the JDK8 image. The sum of all them results in an image of
`143MB` which includes Linux OS, JDK8, PetClinic's code and dependencies' jar.  

> Different Linux image comparison at https://github.com/gliderlabs/docker-alpine#why 


# Memory usage

## PetClinic artifact's memory usage

Run PetClinic application with this command:

```
java -jar target/micronaut-petclinic-0.1.0.BUILD-SNAPSHOT.jar
```

Open `JConsole` (or other profiler such as YourKit) and measure the heap after executing Garbage Collector (GC). The 
result is this:

![jconsole-result](jconsole/result.png)

With no load the application's heap consumption is around `25MB` . However, the memory consumption is bigger than just the
heap, so let's measure it using ``ps`` command:

```
➜  micronaut-docker-study git:(master) ✗ ps aux 79385
USER     PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
ugaitz 79385   0,0  1,2 10285576 195516 s002  S+    5:26PM   0:12.16 java -jar target/micronaut-petclinic-0.1.0.BUILD-SNAPSHOT.jar
```

We can see that PetClinic's process actually is using almost `200MB` of memory.  

## Docker image's memory usage

Run a PetClinic container with this command:

```
docker-compose -f docker-compose-runjava.yml up 
```

Executing ``docker stats`` we can find out how much memory is using the container with no load:

```
CONTAINER ID        NAME                           CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
7cdb35fbbc57        micronaut-docker-study_app_1   0.39%               158.9MiB / 1.952GiB   7.95%               72.8kB / 55.2kB     1.09MB / 0B         14
```

So, the container uses ``159MB`` of memory. 

# Startup time

## PetClinic artifact's startup time
Run PetClinic application with this command:

```
java -jar target/micronaut-petclinic-0.1.0.BUILD-SNAPSHOT.jar
```

```
09:19:22.743 [main] INFO  io.micronaut.runtime.Micronaut - Startup completed in 3649ms. Server Running: http://localhost:8080
```
We can see in app logs that the startup time is 3 649ms.



# Summary

| Feature                                           | Micronaut 1.1.2 |
| ------------------------------------------------- | ----------------- |
| Micronaut App disk usage                          | 36M               |
| Micronaut App disk usage (without dependencies)   | 605KB             |
| Docker Container disk usage                       | 143MB             |
| Micronaut App heap consumption                    | 25MB              |
| Micronaut App memory usage                        | 200MB             |
| Docker container memory usage                     | 159MB             |
| Micronaut App startup time                        | 3 649ms



# Credits

Original PetClinic by https://www.spring.io

Micronaut PetClinic by [Graeme Rocher github repository](https://github.com/graemerocher/micronaut-petclinic)

Docker configuration by https://www.arima.eu

![ARIMA Software Design](https://arima.eu/arima-claim.png)

