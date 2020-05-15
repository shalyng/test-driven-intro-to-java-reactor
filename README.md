---
title: "A Test-driven Intro to Java Reactor"
date: 2020-05-18
author: Oge Nnadi
categories: [Testing, Development]
tags: [java, testing, reactor, reactive, stream, flux, mono]
image: 
image_thumbnail:
image_alt_text:
---

## A Test-driven Intro to Java Reactor
Reactor is [a Java library for parallel programming](https://projectreactor.io/) (it has other esoteric features). As an intro, let us write
a program that pauses for 10 seconds, then performs 3 side effect-heavy computations in parallel. We'll take it in small
steps.

Reactor has 2 main data structures for expressing computations: Mono and Flux. It has others for expressing who cares 
about the result of a computation: a Publisher produces Fluxes and a Subscriber consumes them. A Mono is a stream of 0 or 1 
items and a Flux is a stream of 0 to n items. Think of Reactor streams like Java streams [with a few differences](https://stackoverflow.com/questions/52820232/difference-between-infinite-java-stream-and-reactor-flux).
 
If you can express your parallel computation as a process that produces Monos and Fluxes, Reactor may be a good fit.

### Setup the Reactor environment
To start with Reactor, create a new Maven project in your IDE and place the following snippet in your pom file and you're ready to go

    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
    
        <groupId>org.example</groupId>
        <artifactId>tmp-reactor</artifactId>
        <version>1.0-SNAPSHOT</version>
    
        <!-- Use at least Java 8 -->
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <source>8</source>
                        <target>8</target>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    
        <!-- Set the version for all Reactor dependencies -->
        <dependencyManagement>
            <dependencies>
                <dependency>
                    <groupId>io.projectreactor</groupId>
                    <artifactId>reactor-bom</artifactId>
                    <version>Dysprosium-RELEASE</version> <!-- Search.maven.org for the latest release -->
                    <type>pom</type>
                    <scope>import</scope>
                </dependency>
            </dependencies>
        </dependencyManagement>
    
        <dependencies>
            <!-- Reactor production classes -->
            <dependency>
                <groupId>io.projectreactor</groupId>
                <artifactId>reactor-core</artifactId>
            </dependency>
    
            <!-- Reactor testing classes -->
            <dependency>
                <groupId>io.projectreactor</groupId>
                <artifactId>reactor-test</artifactId>
                <scope>test</scope>
            </dependency>
    
            <!-- Unit tests -->
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter</artifactId>
                <version>5.6.2</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </project>

### Emit 3 numbers sequentially
Now we're ready to code. Let's make a method that produces a Flux of 3 elements sequentially (we'll add more features later).
    
    class SourceTest {
        @Test // 1
        void shouldEmit3Items() {
            StepVerifier.create(new Source().emit()) // 2
                    .expectNextCount(3) // 3
                    .verifyComplete(); // 4
        }
    }
    
    class Source {
        public Flux<Integer> emit() {
            return Flux.just(1, 2, 3); // 5
        }
    } 

1. I'm using JUnit to make it easy to verify that  `emit` behaves correctly. It also makes the code easy to run
from my IDE.

2. StepVerifier is a Reactor class, and a Subscriber for testing Monos and Fluxes. Monos and Fluxes _do nothing_ until
they've been subscribed to. Here we subscribe to `emit` and 

3. Expect to receive any three elements from.

4. Verify that we've reached the end of the stream. 

5. FYI `Flux::just` sends an `onComplete` signal to its subscribers after it is done emitting the elements it is passed.


### Perform a side effect for each number
Let's refine `emit` by making it perform a side effect before emitting each item. It now looks like 
    
    class Source {
        public Flux<Integer> emit() {
            return Flux.just(1, 2, 3)
                    .flatMap(x -> sideEffect().thenReturn(x)); // 1
        }
    
        private Mono<Void> sideEffect() { // 2
            return Mono.fromRunnable(() -> System.out.println(Thread.currentThread().getName())); // 3
        }
    }

1. `flatMap` can take Monos and Fluxes. In this case, it executes a Mono (`sideEffect`) then returns the value it was
passed. `thenReturn` waits for `sideEffect` to finish before returning a value.

2. The return type of `sideEffect` is Mono<Void> meaning that the Mono emits no items (if the return type was, say, 
Mono<Integer> then we could move the thenReturn call in `emit` into `sideEffect`.

3. `Mono.fromRunnable` is a way to make a Mono that can execute a method. Here we print out the name of the thread to
show that everything is running sequentially.  
 
Running the test, `shouldEmit3Items` should show the name of your main thread 3 times in the console:
    
    main
    main
    main

### Perform the side effects in parallel

Let's force the side effects into their own threads. I don't know of a good way to unit-test this behavior so we'll rely
on the console to ensure this works.
    
    class Source {
        public Flux<Integer> emit() {
            return Flux.just(1, 2, 3)
                    .flatMap(x -> sideEffect().thenReturn(x));
        }
    
        private Mono<Void> sideEffect() {
            return Mono.delay(Duration.ofSeconds(0))  // 1
                    .then(Mono.fromRunnable(() -> System.out.println(Thread.currentThread().getName()))); // 2
        }
    }
    
1.  Using `delay` here  is a hack to force the side effect into a different thread. In practice, you'd make, for example, 
a Webflux HTTP or AWS SQS call, which would naturally switch threads.

2. Kind of like `thenReturn`, `then` is used to chain Monos together.

Running  the test `shouldEmit3Items` should now show the names of 3 different threads in the console. Run it
multiple times to see the thread names switch places.
    
    parallel-3
    parallel-2
    parallel-1


### Delay before emitting the first number

What if we wanted to delay for 10 seconds before emitting the first element? This is easily testable

    class SourceTest {
        @Test
        void shouldEmit3ItemsAfterADelay() {
            StepVerifier.create(new Source().emit())
                    .expectSubscription()
                    .expectNoEvent(Duration.ofSeconds(10)) // 1
                    .expectNextCount(3)
                    .verifyComplete();
        }
    }
    
    class Source {
        public Flux<Integer> emit() {
            return Flux.just(1, 2, 3)
                    .delaySequence(Duration.ofSeconds(10)) // 2
                    .flatMap(x -> sideEffect().thenReturn(x));
        }
    
        private Mono<Void> sideEffect() {
            return Mono.delay(Duration.ofSeconds(0))
                    .then(Mono.fromRunnable(() -> System.out.println(Thread.currentThread().getName())));
        }
    }
    
1. `expectNoEvent` expects that no events are signaled to the Subscriber. Examples of events are `onNext`, and `onComplete`
and `onSubscribe`. Yes, subscribing to a Mono/Flux triggers an event, so if you omit `expectSubscription` on the line
above, this test will fail.

2. `delaySequence` adds a delay before emitting the first element in the Flux.

Our newly renamed test, `shouldEmit3ItemsAfterADelay`, gets annoying to run since we have to wait for 10 seconds to see
it pass. We can get rid of this delay by modifying the test
    
    class SourceTest {
        @Test
        void shouldEmit3ItemsAfterADelay() {
            StepVerifier.withVirtualTime(() -> new Source().emit()) // 1
                    .expectSubscription()
                    .expectNoEvent(Duration.ofSeconds(10))
                    .expectNextCount(3)
                    .verifyComplete();
        }
    }


 Sadly, this causes the console output to show all the side effects running on the same thread
     
     main
     main
     main

`withVirtualTime` works by switching to a single thread and then manipulating the system clock. We can regain the
multithreaded behavior even in this test with some extra hacking, but the test suffices for now so we'll end here.

To learn more about Reactor, read 

* [The Reactor docs](https://projectreactor.io/docs/core/release/reference/)
* [The Reactive Manifesto's Java spec](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.3/README.md) of
which Reactor is an implementation.
* Understand how Reactor code in the wild works ([e.g. sample AWS Kinesis code](https://github.com/awsdocs/aws-doc-sdk-examples/blob/master/javav2/example_code/kinesis/src/main/java/com/example/kinesis/KinesisStreamReactorEx.java)),
 and
* Talk to your local Java guru

Thanks to Travis Klotz for sharing his Reactor expertise with me.
