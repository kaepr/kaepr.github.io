---
date: '2024-12-18T22:07:00+05:30'
draft: false
title: 'Setup Clojure with GraalVM for Native Image'
tags: [clojure, graalvm]
description: "Setup Clojure, GraalVM for generating native images. Also add Flowstorm for debugging."
---

This post will detail the steps to setup Clojure and GraalVM to generate native executables. I used a similar approach when creating my project [cljcc](https://github.com/kaepr/cljcc). It has a few extra steps on on top of generating a native image, but this 
post will have just the minimum things required to build uberjar and generating image.

## Requirements

* [GraalVM](https://www.graalvm.org/latest/getting-started/)
* [Clojure](https://clojure.org/guides/getting_started)
    * [tools.build](https://clojure.org/guides/tools_build)
    * [clj-easy graal build time](https://github.com/clj-easy/graal-build-time)

## Project Structure 

```
├── build.clj
├── deps.edn
├── src
│   └── demo
│       └── core.clj
└── target 
```


The `demo/core.clj` file simply prepends `"Hello"` to the first argument, and prints to stdout.

```clj
(ns demo.core
  (:gen-class))

(defn -main [& args]
  (println (str "Hello " (first args))))

```

The `:gen-class` is necessary, as this informs the [build process](https://clojure.org/reference/compilation) to generate a Main entrypoint for the Java program. Without `:gen-class`, although a uberjar will be built, 
the native image generation will fail with the below error. 

```
Error: Main entry point class 'demo.core' neither found on
classpath: '/home/shagun-agrawal/Development/setup-clj-graalvm/target/demo.core-1.0.0-standalone.jar' nor
modulepath: '/home/shagun-agrawal/.sdkman/candidates/java/23-graalce/lib/svm/library-support.jar'.
Internal exception: com.oracle.svm.core.util.UserError$UserException: Main entry point class 'demo.core' neither found on
classpath: '/home/shagun-agrawal/Development/setup-clj-graalvm/target/demo.core-1.0.0-standalone.jar' nor
modulepath: '/home/shagun-agrawal/.sdkman/candidates/java/23-graalce/lib/svm/library-support.jar'.
	at org.graalvm.nativeimage.builder/com.oracle.svm.core.util.UserError.abort(UserError.java:85)
	at org.graalvm.nativeimage.builder/com.oracle.svm.hosted.NativeImageGeneratorRunner.buildImage(NativeImageGeneratorRunner.java:440)
	at org.graalvm.nativeimage.builder/com.oracle.svm.hosted.NativeImageGeneratorRunner.build(NativeImageGeneratorRunner.java:711)
	at org.graalvm.nativeimage.builder/com.oracle.svm.hosted.NativeImageGeneratorRunner.start(NativeImageGeneratorRunner.java:139)
	at org.graalvm.nativeimage.builder/com.oracle.svm.hosted.NativeImageGeneratorRunner.main(NativeImageGeneratorRunner.java:94)
```

Below is the `deps.edn` file. It has an extra alias for nrepl. 

```edn
{:paths ["src"]
 :deps {com.github.clj-easy/graal-build-time {:mvn/version "1.0.5"}}
 :aliases
 {:run-main {:main-opts ["-m" "demo.core"]}
  :build {:deps {io.github.clojure/tools.build {:git/tag "v0.10.6" :git/sha "52cf7d6"}}
          :ns-default build}
  :nrepl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.0"}
                       cider/cider-nrepl {:mvn/version "0.50.2"}
                       refactor-nrepl/refactor-nrepl {:mvn/version "3.10.0"}}
          :main-opts ["-m" "nrepl.cmdline" "--interactive" "--color" "--middleware" "[cider.nrepl/cider-middleware,refactor-nrepl.middleware/wrap-refactor]"]}}}

```

Adding aliases to `deps.edn` can be managed using a tool called [neil](https://github.com/babashka/neil). 
I found about this tool from [Developer Tooling for Speed and Productivity in 2024 | Vedang Manerikar](https://www.youtube.com/watch?v=pVvuyaRDA58). 
I earlier used to rely on Doom Emacs `cider-jack-in-clj` function, which starts a clojure REPL automatically and connects to it, but I wasn't aware of how it works ( for e.g. the command lines options being passed etc ). 
Starting up a repl in different shell and connecting to it from my editor is much simpler. 
It also makes it editor agnostic, as the setup for starting a REPL is present in the deps file itself

The above video also includes setup for logging, flowstorm debugger, documentation, project structure etc.

Use `clj -M:nrepl` to start the server. I then use Doom Emacs `cider-connect` function to attach to it.


## Building Uberjar

`build.clj` 

```clj
(ns build
  (:require [clojure.tools.build.api :as b]))

(def lib 'demo.core)
(def version "1.0.0")
(def class-dir "target/classes")
(def uber-file (format "target/%s-%s-standalone.jar" (name lib) version))

;; delay to defer side effects (artifact downloads)
(def basis (delay (b/create-basis {:project "deps.edn"})))

(defn clean [_]
  (b/delete {:path "target"}))

(defn uber [_]
  (clean nil)
  (b/copy-dir {:src-dirs ["src" "resources"]
               :target-dir class-dir})
  (b/compile-clj {:basis @basis
                  :ns-compile '[demo.core]
                  :class-dir class-dir})
  (b/uber {:class-dir class-dir
           :uber-file uber-file
           :basis @basis
           :main 'demo.core}))
```

Run `clj -T:build uber`, which generates a jar file under `/target` directory.

```shell
java -jar ./target/demo.core-1.0.0-standalone.jar World

Hello World
```

To automate the creation of a new application, take a look at [deps-new](https://github.com/seancorfield/deps-new). It automatically sets up a `deps.edn` and `build.clj` file. and has aliases for starting repl, building uberjar etc.

The `deps.edn` file also has a dependency on [graal-build-time](https://github.com/clj-easy/graal-build-time). Adding it to deps adds this library to classpath.

```shell
clj -Spath # without adding library in deps.edn
src:/home/shagun-agrawal/.m2/repository/org/clojure/clojure/1.12.0/clojure-1.12.0.jar:/home/shagun-agrawal/.m2/repository/org/clojure/core.specs.alpha/0.4.74/core.specs.alpha-0.4.74.jar:/home/shagun-agrawal/.m2/repository/org/clojure/spec.alpha/0.5.238/spec.alpha-0.5.238.jar

clj -Spath # after adding library in deps.edn
src:/home/shagun-agrawal/.m2/repository/com/github/clj-easy/graal-build-time/1.0.5/graal-build-time-1.0.5.jar:/home/shagun-agrawal/.m2/repository/org/clojure/clojure/1.12.0/clojure-1.12.0.jar:/home/shagun-agrawal/.m2/repository/org/clojure/core.specs.alpha/0.4.74/core.specs.alpha-0.4.74.jar:/home/shagun-agrawal/.m2/repository/org/clojure/spec.alpha/0.5.238/spec.alpha-0.5.238.jar
```

Without this dependency, the generated jar after the build also has differences.
```shell
jar tf demo.core-1.0.0-standalone.jar | grep '/' | cut -d'/' -f1 | sort -u # without dep
clojure
demo
META-INF

jar tf demo.core-1.0.0-standalone.jar | grep '/' | cut -d'/' -f1 | sort -u # with dep
clj-easy
clj_easy
clojure
demo
META-INF
```

The native image commands needs to initialize `.class` files at build time. To automatically identify which files needs to be initialized, 
`graal-build-time` library will detect `.class` files, and uses the feature flag ( mentioned in the below command ) to mark them to be initialized at build time.

## Generate Native Image

```shell
native-image -jar target/demo.core-1.0.0-standalone.jar -o target/demo -H:+ReportExceptionStackTraces --features=clj_easy.graal_build_time.InitClojureClasses --report-unsupported-elements-at-runtime --verbose --no-fallback
```

This generates an executable at `/target/demo`.

```shell
./target/demo "World"

Hello World
```


## Alias for adding Flowstorm

I couldn't find the command in neil which adds flowstorm alias to a project. The below alias will setup nREPL and flowstorm.

```edn
:storm {;; for disabling the official compiler
          :classpath-overrides {org.clojure/clojure nil}
          :extra-deps {io.github.clojure/tools.build {:mvn/version "0.10.3"}
                       nrepl/nrepl {:mvn/version "1.3.0"}
                       cider/cider-nrepl {:mvn/version "0.50.2"}
                       refactor-nrepl/refactor-nrepl {:mvn/version "3.10.0"}
                       com.github.flow-storm/clojure {:mvn/version "1.11.4-1"}
                       com.github.flow-storm/flow-storm-dbg {:mvn/version "4.0.1"}}
          :jvm-opts ["-Dclojure.storm.instrumentEnable=true"
                     "-Dclojure.storm.instrumentOnlyPrefixes=<NAMESPACE>"]
          :main-opts ["-m" "nrepl.cmdline" "--interactive" "--color" "--middleware" "[flow-storm.nrepl.middleware/wrap-flow-storm,cider.nrepl/cider-middleware,refactor-nrepl.middleware/wrap-refactor]"]}
```

Use `clj -M:storm` to start a nREPL session with flowstorm. Evaluate `:dbg` in the REPL to launch the debugger. 
Refer [Flowstorm Documentation](https://flow-storm.github.io/flow-storm-debugger/user_guide.html) on how to use the debugger.

## References

- [Developer Tooling for Speed and Productivity in 2024 | Vedang Manerikar](https://www.youtube.com/watch?v=pVvuyaRDA58)
- [Class Initialization in Native Image](https://www.graalvm.org/latest/reference-manual/native-image/optimizations-and-performance/ClassInitialization/)
- [Clojure for the Brave and True, Working with JVM](https://www.braveclojure.com/java/)

