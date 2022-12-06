---
title: "Serverless cold start for the rest of us"
date: 2022-12-05T12:00:00+0100
draft: false
github_link: "https://github.com/claranet-ch/java-lambda-optimization"
author: "Celestino"
tags:
- Java
- AWS Lambda
- example
image: /images/cold_start/taylor-vick-M5tzZtFCOfs-unsplash.jpg
description: ""
toc:
---
<div class="text-center mb-3">
    <small>Photo by <a href="https://unsplash.com/@tvick?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Taylor Vick</a> on <a href="https://unsplash.com/s/photos/network?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a></small>
</div>

## Introduction

Java is one of the most popular platforms available today. Despite its age (27 years old) it continues to offer one of the most performant runtime platforms available on the market.
Its grammar can be perceived as _verbose_, its _strongly typed_ nature _difficult_ for newbies, but if you get past that you can leverage the JVM (Java Virtual Machine), a platform that:

- Constantly improves its efficiency and performance at runtime, thanks to continuous profiling and its multi-stage <abbr title="Just in Time">JIT</abbr> compiler
- Is available on virtually all possible (server) hardware/architectures out there
- Has a strict “backwards compatibility contract”: the latest version of the JVM can still run bytecode compiled 27 years ago, at least in theory
- Has a vibrant community: you can always find an User Group (JUG) close to your location and leverage a huge ecosystem of OSS libraries/frameworks/applications

These factors have made Java one of the most used platforms in corporate environments, running applications in almost every sector. So, if you work for a big company, there’s a high chance that you are writing Java code or that you’re running your code on a JVM.

## The Challenge with Serverless Functions

However, the very thing that makes the Java runtime so performant over time for server applications, the <abbr title="Just in Time">JIT</abbr> compiler, is the reason why a Java program might be too slow to start in a Serverless environment.  
The JVM needs to complete a warm-up phase where the bytecode is inspected multiple times, optimized, translated to hardware-specific instructions, and finally run.

This process might take _seconds_ to complete, a very long time in the context of Microservices or Serverless functions.

## <abbr title="Ahead of Time">AOT</abbr> to the rescue?

There’s an ongoing effort both within the JVM project ([Project Leyden](https://blogs.oracle.com/javamagazine/post/java-leyden-static-images) and [GraalVM](https://www.graalvm.org/latest/reference-manual/native-image/)) specifically to address startup times for short-lived process, like Serverless functions.

Problem solved?
Yes, absolutely. If your stack is compatible with AOT and native image, by all means GO FOR IT.

But that’s not always the case. Let me clarify that with an example.

### Hello, SOAP (jax-ws) client

Let's assume that:

- You work for a big company, with rules and processes in place concerning which technologies/libraries/frameworks can be used.
- You maintain a monolithic Java application _Application A_, whose code has been recently migrated to Java 11. This application is a billing system, which validates the VAT Numbers of EU-based commercial customers using the [VIES SOAP WebService](https://ec.europa.eu/taxation_customs/vies/#/vat-validation). The implementation is done using JAX-WS
- _Application A_ is deployed on AWS

There is now another application being developed in your organization - _Application B_, a web shop - which needs to perform the same validation.
In order to avoid increased traffic to _Application A_ and to better handle sudden traffic spikes you decide to extract the logic and create an AWS Lambda function.

In this particular scenario, generating a native image can be [really tricky](https://github.com/oracle/graal/issues/2188) so plain Java is the only viable option. We have to deal with cold starts on our own.

Note: This example is an abstraction of real experiences that I had while working for different customers; what follows is my personal journey in analyzing "cold start" problems.
I wanted to share it with you, and with my "future self" in the hope it might be useful, and to get some feedback.

## Default Corretto 11 runtime

Let's start from the default runtime provided by AWS: Corretto 11

### 1. Optimize your code

Make sure to initialize expensive (thread-safe) resources outside the `handleRequest` method, in order to reuse resources for subsequent invocations

```java
public class Handler implements RequestHandler<ValidationRequest, ValidationResponse> {

    private static final Logger LOGGER = LoggerFactory.getLogger(Handler.class);
    private static final CheckVatPortType PORT;

    // initialize PORT in the static initializer
    static {
        var wsdlLocation = Thread.currentThread().getContextClassLoader()
            .getResource("wsdl/checkVatService.wsdl");
        var qName = new QName("urn:ec.europa.eu:taxud:vies:services:checkVat", "checkVatService");
        PORT = new CheckVatService(wsdlLocation, qName).getCheckVatPort();
    }

    public Handler() {
    }

    @Metrics(captureColdStart = true)
    @Tracing
    @Override
    public ValidationResponse handleRequest(ValidationRequest input, Context context) {
        // validates input params and calls performValidation
    }

    @Tracing(segmentName = "Call remote webservice")
    private static ValidationResponse performValidation(String country, String vatNumber) {
        var valid = new Holder<Boolean>();
        var name = new Holder<String>();
        var address = new Holder<String>();
        // PORT is already initialized
        PORT.checkVat(
            new Holder<>(country),
            new Holder<>(vatNumber),
            new Holder<>(),
            valid,
            name,
            address
        );
        return new ValidationResponse(valid.value, name.value, address.value);
    }
}
```

### 2. Optimize your runtime

If your code is very simple (like the one above) you can decide to disable _incremental JIT compilation_, also known as _Tiered Compilation_.
By doing so we instruct the JVM to ignore optimizations. The lambda will never reach peak performances at runtime, but at the same time we'll get a quicker startup.
This is a good compromise in this case, as the task is very simple and short-lived by design.

To do that, you can set the following environment variable in the lambda configuration:

```shell
JAVA_TOOL_OPTIONS="-XX:+TieredCompilation -XX:TieredStopAtLevel=1 -XX:+UseSerialGC"
```

Read the [official AWS blog post](https://aws.amazon.com/blogs/compute/optimizing-aws-lambda-function-performance-for-java/) for more information.

### 3. Results

With all this optimization in place, we can now measure a cold start, sending the following _test_ event to the lambda `vies-proxy-11`:

```json
{
  "country": "IE",
  "vatNumber": "10000"
}
```

![Cold start - JRE 11 default](/images/cold_start/001_vies-proxy-11-base.png)

- Init duration: `1.57s`
- Duration: `800ms`

Not that bad as starting point, considering that we have allocated just 512MB of RAM.

But **we have to do better**, since the lambda execution time will have a direct impact on the end-user experience.



## JRE 19 (Docker runtime)

Recently the JVM dev team put a lot of effort into reducing startup time of Java programs, so it makes definitely sense to run our lambda with the latest (stable) runtime.   
AWS Lambda supports running custom Docker images, so let's go ahead and try to build a custom image.
You can find the full Dockerfile [here](https://github.com/claranet-ch/java-lambda-optimization/blob/main/docker/jdk19/Dockerfile).

### Multi-stage is better

We'll build a [multi-stage](https://docs.docker.com/build/building/multi-stage/) Dockerfile.
The resulting image will have fewer layers and therefore will be more compact in size, resulting in a reduced cold start.

### 1. Build a smaller JRE

JDK9 introduced `jlink`, a tool that allows us to build a smaller version of the JRE. We can include only the modules we need:

```dockerfile
# Find JDK module dependencies dynamically from the uber jar
RUN jdeps -q \
    --ignore-missing-deps \
    --multi-release 19 \
    --print-module-deps \
    software/target/function.jar > /tmp/jre-deps.info

# Create a slim JRE which only contains the required modules to run the function
RUN jlink --verbose \
    --compress 2 \
    --strip-java-debug-attributes \
    --no-header-files \
    --no-man-pages \
    --output /jre-slim \
    --add-modules $(cat /tmp/jre-deps.info)
```

### 2. Use (App) Class-Data Sharing

#### Class-Data sharing (CDS)

Since version 12, the JVM ships with a pre-generated [Shared Archive](https://docs.oracle.com/en/java/javase/19/vm/class-data-sharing.html) of system classes.
This archive contains information ready to be used by the JVM, reducing CPU usage, memory usage, and startup time.
Since we've generated a trimmed version of the JRE, we need to regenerate the Shared Archive:

```dockerfile
# Generate CDS archive for our slim JRE
# It creates the file /jre-slim/lib/server/classes.jsa
RUN /jre-slim/bin/java -Xshare:dump -XX:+UseSerialGC -version
```

#### AppCDS

Since version 13 it's possible to generate such archive also for the application code. Every millisecond counts for us, so we'll do it:

```dockerfile
RUN export AWS_LAMBDA_RUNTIME_API="localhost:8080" &&\
    /jre-slim/bin/java --add-opens java.base/java.util=ALL-UNNAMED -XX:ArchiveClassesAtExit=appCds.jsa -jar function.jar "com.claranet.vies.proxy.Handler::handleRequest"  2>/dev/null 1>&2 || :
```

the process will exit with an error (ignored by the command), but the important thing for us is to process as many classes as we can.

### 3. Results

Let's run the `vies-proxy-19` function with the same payload as before

![Cold start - JRE 19 docker](/images/cold_start/002_vies-proxy-19-docker.png)

![Tracing - JRE 19 docker](/images/cold_start/003_vies-proxy-19-docker-tracing.png)

- Init duration (from Tracing) : `3.72s`
- Duration: `9.6s`

wow, that's disappointing. However, if we increase allocated RAM to `2048MB` numbers improve:

- Init duration (from Tracing) : `2.27s`
- Duration: `3.67s`

~60% improvement compared to the `512MB` configuration, but still more than the default JVM. Clearly this is not the way to go.

## JRE 19 (custom runtime)

It is also possible to run our JRE 19 as a custom runtime. This will remove the overhead of running another virtualized environment.
We can reuse the content of the Dockerfile to create our custom runtime. 

See https://github.com/claranet-ch/java-lambda-optimization/blob/main/infrastructure/src/main/java/com/claranet/vies/proxy/stack/ViesProxyStack.java#L79-L105

> Custom runtimes are only available on `x86_64` CPU architecture

### Results

Let's run the `vies-proxy-19-custom` function with the same payload:

![Cold start - JRE 19 custom runtime](/images/cold_start/004_vies-proxy-19-custom.png)

- Init duration (from Tracing) : `996ms`
- Duration: `2.8s`

Eureka! We've got sub-second cold start! :tada:

For the record, increasing the allocated RAM to `2048MB` we get:

- Init duration (from Tracing) : `852ms`
- Duration: `972ms`

Way better, but maybe we try something else to reduce startup times even more. 

## AWS SnapStart

Very recently, AWS [announced SnapStart](https://aws.amazon.com/blogs/aws/new-accelerate-your-lambda-functions-with-lambda-snapstart/) at re:Invent.

SnapStart performs a _snapshot_ of the micro-container running the lambda function and then restores it. 

> SnapStart is only available for *x86_64* CPU architecture and might not be compatible with your code. Please check https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html for additional information

### 1. Application initialization

Now we can perform a snapshot of the running JVM. So we can also get rid of the application initialization time, right?

Let's take a closer look at the tracing for the latest invocation of `vies-proxy-19-custom`:

![SOAP client initialization cost](/images/cold_start/005_initialization-cost.png)

as you can see the Application initialization takes more than `2s` (2.86 - 0.563). This might have something to do with the SOAP client initialization and the proxies it generates to generate the request and parse the response.

Let's modify the code to try to include these objects in the snapshot as well.

### 2. Lifecycle hooks

SnapStart supports [CRaC](https://docs.aws.amazon.com/lambda/latest/dg/snapstart-runtime-hooks.html) checkpoint API, so let's modify the Lambda Handler accordingly:

```java
// [...]
import org.crac.Core;
import org.crac.Resource;
// [...]
public class Handler implements RequestHandler<ValidationRequest, ValidationResponse>, Resource {

    public Handler() {
        Core.getGlobalContext().register(this); // <-- register snapshot listener
    }
    
    // [...]

    @Override
    public void beforeCheckpoint(org.crac.Context<? extends Resource> context) {
        // as per https://docs.aws.amazon.com/lambda/latest/dg/snapstart-runtime-hooks.html SnapStart supports CRaC hooks.
        // We use that to load as many objects as we can in memory before performing the checkpoint,
        // in order to prevent expensive initializations from happening at runtime
        performValidation("IE", "123456"); // <-- forces SOAP client initialization
    }

    @Override
    public void afterRestore(org.crac.Context<? extends Resource> context) {
        LOGGER.info("Restore complete");
    }
}    
```

full source code available [here](https://github.com/claranet-ch/java-lambda-optimization/blob/main/software/src/main/java/com/claranet/vies/proxy/Handler.java)

### 3. Results

After enabling SnapStart and publishing a new version, let's run our usual test:

![Cold start - SnapStart](/images/cold_start/006_snapstart.png)

- Restore duration : `591ms`
- Duration: `226ms`

Sub-second response time! Outstanding! :tada:

For reference, not optimized code (i.e. without snapshot listener) results in:
- Restore duration : `331ms`
- Duration: `614ms`
 
While increasing allocated RAM to `2048MB` results in:
- Restore duration : `275ms`
- Duration: `238ms`


## Conclusions

Cold start can be really an issue for user-facing Java serverless functions. There are some promising technologies which you can already use, but they cannot work in all cases. 

If you can go _native_ using AOT and GraalVM, then GO FOR IT, as it is probably the best available option right now.  

If you can't, I hope that this post can give you some hints and a blueprint for a process you can follow.

If you want to experiment yourself with the code, you can find it here: https://github.com/claranet-ch/java-lambda-optimization
