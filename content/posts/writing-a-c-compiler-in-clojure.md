---
date: "2024-12-21"
draft: true
title: 'Writing a C Compiler in Clojure'
summary: "This post describes my experience of writing a C compiler in Clojure."
description: "This post describes my experience of writing a C compiler in Clojure."
toc: true
tags: ["clojure", "compilers"]
hidePagination: true
showTags: true
readTime: true
autonumber: true
---


I recently found the book [Writing a C Compiler](https://nostarch.com/writing-c-compiler) by [Nora Sandler](https://norasandler.com/). The book goes step by step in implementing a subset of C.
New language features are added  in each chapter, and at the end of each chapter there are [tests](https://github.com/nlsandler/writing-a-c-compiler-tests) which verify that behavior. The tests are extensive and also heavily commented, which help in debugging.
The book is fun to read and implement. All features / steps are explained in detail, with ample pseudcode and explanations, but also not hand holding.
All the algorithms/implementation is in a pseudocode / pythonesque language, but the author has also shared their reference implementation in [OCaml](https://github.com/nlsandler/nqcc2).

I have been wanting to make something in Clojure for a while. My first project with Clojure was a [Nintendo Entertainment System emulator](https://github.com/kaepr/nes), but after finishing the CPU, I got pretty burnt out.
I had made an [emulator before](https://github.com/kaepr/gameboy_emulator) (Gameboy), and it wasn't different enough and I slowly lost interest. Also performing lot of bit manipulations / operations in Clojure was not fun.

Apart from that I did some coding challenges like Advent of Code etc, but otherwise did not create any application using Clojure.
I had earlier seen this presentation [What's So Hard About Writing A Compiler, Anyway? Oh - Ramsey Nasser
](https://www.youtube.com/watch?v=_7sncBhluXI), which shared their general approach of making a compiler written in Clojure.
This talk motivated me to pursue writing the project in Clojure. Working with an AST, which would just be map of data felt like the perfect thing to build with Clojure.

I have always had interest in compilers. Before this, I had completed [Crafting Interpreters](https://craftinginterpreters.com/).
I have completed [Programming Languages course by Dan Grossman](https://www.coursera.org/instructor/~873260), which explains and implements concepts (such as variables, functions, type systems) in various different languages.
These resources definitely helped me out, as I had prior familiarity with the concepts presented in this book, but even without that I feel the book explains everything for somebody with no prior experience.
I had not written a compiler for C like language, and was not aware of how x86 assembly gets generated. This book turned out be a interesting project, and I learned a lot about C and x86 assembly.

I have implemented the first 12 chapters from the book. The compiler supports the below features. (Each feature roughly corresponds to each chapter in the book).

- Operators
    - Unary `-, ~`
    - Binary `+, -, *, /, %, <<, >>`
    - Relational `<, >, >=, <=, == !=`
    - Logical `!, &&, ||`
- Variables (with storage class specifiers)
- If/else statements
- Loops
- Functions
- Types (`int, unsigned int, long, unsigned long`)

This post details my experience writing the compiler. It mentions the problems and what approach I took to solve them.
There are a lot of references to other projects/conference talks/articles throughout, which gave me ideas on how to refactor / make it simpler etc.
The post will be broken down into the major passes of the compiler.
Each subsequent pass takes in as input the return value from the last stage.

The code is avaiable at [kaepr/cljcc](https://github.com/kaepr/cljcc).


```clojure
(defn run [source]
  (-> source
      lex
      parse
      tacky
      analyze
      assembly
      emit))
```

## Lexer



## Parser




## Analyze

One thing which in general irked me throughout was whether I am doing things `clojure way`.
There is a section in this talk, [Learning and Teaching Clojure on the job at Amperity - Dave Fetterman
](https://youtu.be/QBsjYyg9bLE?si=o0DXPxD0X51SXduv&t=287), where the speaker mentions that they were writing Lisp, basically like C. Below is a quote from the speaker,

> I found something called the let statement, this let me make it (lisp) look just like C, variable thing, variable thing, oh I have an imperative program with parentheses.

I felt the same way. There were certain instances where I felt I had come up with a neat functional way of doing things, but lot of my code, or the surrounding code still felt imperative.
Some functions became huge and unwieldy, but I wasn't really getting how should I refactor it / make it simple.

For example, below is the code for typechecking a local variable declaration.

```clj
(defn- typecheck-local-scope-variable-declaration
  [{:keys [identifier variable-type storage-class initial] :as d} ident->symbol]
  (condp = storage-class
    :extern (let [_ (when (not (nil? initial))
                      (exc/analyzer-error "Initializer on local extern variable declaration." d))
                  prev-symbol (get ident->symbol identifier)
                  prev-type (:type prev-symbol)
                  _ (when (and prev-symbol (not= prev-type variable-type))
                      (exc/analyzer-error "Redeclared with different types." {:declaration1 d
                                                                              :declaration2 prev-symbol}))
                  symbols (if prev-symbol
                            ident->symbol
                            (assoc ident->symbol
                                   identifier
                                   (sym/create-symbol variable-type (sym/static-attribute (sym/no-initializer-iv) true))))]
              {:declaration d
               :ident->symbol symbols})
    :static (let [initial-value (to-static-init initial variable-type)
                  updated-symbols (assoc ident->symbol
                                         identifier
                                         (sym/create-symbol variable-type (sym/static-attribute initial-value false)))]
              {:declaration d
               :ident->symbol updated-symbols})
    (let [updated-symbols (assoc ident->symbol
                                 identifier
                                 (sym/create-symbol
                                  variable-type
                                  (sym/local-attribute)))
          casted-e (if (nil? initial)
                     initial
                     (convert-to-exp initial variable-type))
          t-e (typecheck-optional-expression casted-e updated-symbols)]
      {:declaration (assoc d :initial t-e)
       :ident->symbol updated-symbols})))
```

I could refactor out some functionality above, such as the `:static` and `:extern` cases can be in their different functions, but it does not help that much.
I would be hiding these inside some other function, and sometimes I just prefer the entire logic to be present at once.
Another problem was naming, the original function name was already too long, and another specialized case function felt wrong.

There are a lot of such instances, especially in this pass of the compiler, where I want to refactor.
There seems to be a lot of repetitive code, but it's mixed with the complexity of C specification aswell (for e.g. storage class specifiers), so I still haven't been able to refactor it to something I am happy with.

One of the solutions to this problem ( or in general ), I guess would be to be read more code.
I have watched a lot of conference talks, but throughout building this I realized I haven't actually read more codebases.
One of the projects which helped me and gave new ideas is [tools.analyzer](https://github.com/clojure/tools.analyzer). I will describe how it helped in the [tacky](#tacky) section.

Listed below are some articles, projects which can help to find other projects, ideas etc for implementing things in `clojure way`.

- [Which Clojure codebases should I read? How and why? by Aditya Athalye](https://www.evalapply.org/posts/which-clojure-codebases-to-read-how-and-why/index.html#main)
- [A Clojure view of "Mars Rover" by Aditya Athalye](https://www.evalapply.org/posts/clojure-mars-rover/index.html#main) Models the mar's rover problem kepeing clojure's it's just data ideology
- [Timothy Baldridge - Data All The ASTs](https://youtu.be/KhRQmT22SSg?si=Mu-Jw-1fEm12Od6N) Clojure way of processing ASTs
- [Clojure Design Patterns by Mykhailo Kozik](https://mishadoff.com/blog/clojure-design-patterns/)




### Resolve

### Label Loop

### Typecheck



## Tacky

## Assembly

## Emission


I am yet to complete the book. Many of the upcoming features are quite core to C, such as pointers, arrays, structs.
As of now, I feel the base structure of the compiler is there, and each feature can be neatly added on top.
But as those features are more complex than the ones presented above, there is a still a gradual difficulty curve,
and I notice more oppurtunities to make my code simpler.
Looking forward to implementing the rest of the book !
