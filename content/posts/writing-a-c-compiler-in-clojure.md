---
date: "2024-12-24"
draft: false
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
New language features are added in each chapter, and at the end of each chapter there are [tests](https://github.com/nlsandler/writing-a-c-compiler-tests) which verify that behavior. The tests are extensive and also heavily commented, which help in debugging.
The book is fun to read and implement. All features / steps are explained in detail, with ample pseudcode and explanations, but also not hand holding.
All the algorithms/implementation is in a pseudocode / pythonesque language, but the author has also shared their reference implementation in [OCaml](https://github.com/nlsandler/nqcc2).

I have been wanting to make something in Clojure for a while. My first project with Clojure was a [Nintendo Entertainment System emulator](https://github.com/kaepr/nes), but after finishing the CPU, I got pretty burnt out.
I had made an [emulator before](https://github.com/kaepr/gameboy_emulator) (Gameboy), and it wasn't different enough and I slowly lost interest. Also performing lot of bit manipulations / operations in Clojure was not fun.

Apart from that I did some coding challenges like Advent of Code etc, but otherwise did not create any application using Clojure.
I had earlier seen this presentation [What's So Hard About Writing A Compiler, Anyway? Oh - Ramsey Nasser
](https://www.youtube.com/watch?v=_7sncBhluXI), which shared their general approach of making a compiler written in Clojure.
This talk motivated me to pursue writing the project in Clojure. Working with an AST, which would just be map of data felt like the perfect thing to build with Clojure.

I have always had interest in compilers. Before this, I had completed [Crafting Interpreters](https://craftinginterpreters.com/).
I have completed [Programming Languages course by Dan Grossman](https://www.coursera.org/instructor/~873260), which explains and implements concepts (such as variables, functions, type systems) in different types of languages.
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
The post is sectioned into the major passes of the compiler.
Each subsequent pass takes in as input the return value from the last stage.

The code is available at [kaepr/cljcc](https://github.com/kaepr/cljcc).


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

The first stage is the lexer, which converts the source input file to a list of tokens.

```clj
(def source "
int main(void) {
  return 42;
}
")

(lex source)

;; => 
{:tokens
 [{:kind :kw-int, :line 2, :col 1}
  {:kind :identifier, :line 2, :col 5, :literal "main"}
  {:kind :left-paren, :line 2, :col 9}
  {:kind :kw-void, :line 2, :col 10}
  {:kind :right-paren, :line 2, :col 14}
  {:kind :left-curly, :line 2, :col 16}
  {:kind :kw-return, :line 3, :col 3}
  {:kind :number, :line 3, :col 10, :literal "42"}
  {:kind :semicolon, :line 3, :col 12}
  {:kind :right-curly, :line 4, :col 1}
  {:kind :eof, :line 5, :col 1}],
 :line 5,
 :col 1}
```

The lexer is responsible for identifying keywords, numbers, identifiers, operators, ignoring whitespace etc.
Each character from the input file needs to be match for a valid token. 
For e.g., if the current character is `\n`, we can ignore it and move to the next character.
If it's a valid alphabetical character, then we need to parse the remaining characters till we encounter a word break, then we check whether it's a valid keyword or an identifier. 
Similar logic is present to tokenize operators, numbers etc.

My initial approach was using an index into the input string, which would point to current character.
I would try to match this character, and from there decide whether to parse it as a number, identifier etc. This is how it looked like.


```clj
(defn lex [idx source]
  (let [ch (nth source idx)]
    (cond
      (digit? ch) (let [end-idx (; use regex to find word break
                                 ; return that index
                                 )
                        digit (subvec source idx end-idx)
                        _ (valid-digit? digit)
                        token (create-digit-token digit)]
                    (conj (lex (inc end-idx) source) token))
      (alphabet? ch) (let [end-idx (;use regex to find word break
                                    ;return that index
                                    )
                           identifier (subvec source idx end-idx)
                           keyword? (is-keyword? identifier)
                           token (create-token identifier keyword?)]
                       (conj (lex (inc end-idx) source) token))
      (...) (...)
      :else (lexer-error idx source))))

```

But this became too complicated.
I was heavily using the index.
In each condition clause, I would try to find a corresponding ending index ( after matching letters / digits etc ).
From both those indexes, would get the identifier and create the token.
I haven't added the code for handling errors above, so each condition was more complex. 
I was getting off by one errors and it was hard to debug. 
This approach was similar to one present in [Crafting Interpreters | Scanner](https://craftinginterpreters.com/scanning.html). 
I was converting imperative Java code to Clojure, and it wasn't working out.

I had earlier watched this video [Code Review: Clojure Lexer by ThePrimeAgen](https://www.youtube.com/watch?v=SGBuBSXdtLY). 
In that video, they were doing code review, for a lexer written in Clojure.
[Clojure Lexer implementation by Vikasg7](https://github.com/ThePrimeagen/ts-rust-zig-deez/tree/master/clj)

This lexer was also written for a C like language, thus the main structure for the lexer was also suitable for tokenizing C programs. 
I took reference from this code, and implemented the lexer.

```clj
(defn- lexer-ctx []
  {:tokens []
   :line 1
   :col 1})

(defn lex
  ([source]
   (lex source (lexer-ctx)))
  ([[ch pk th :as source] {:keys [line col] :as ctx}]
   (cond
     (empty? source) (update ctx :tokens #(conj % (t/create :eof line col)))
     (...) (...)
     (whitespace? ch) (recur (next source)
                             (-> ctx
                                 (update :col inc)))
     (letter? ch) (let [[chrs rst] (split-with letter-digit? source)
                        lexeme (apply str chrs)
                        cnt (count chrs)
                        kind (t/identifier->kind lexeme)
                        token (if (= :identifier kind)
                                (t/create kind line col lexeme)
                                (t/create kind line col))]
                    (recur (apply str rst) (-> ctx
                                                    (update :col #(+ % cnt))
                                                    (update :tokens #(conj % token)))))
     :else (exc/lex-error {:line line :col col}))))
```

The main insights which helped simplify was that keeping an index around is not necessary.
I used index to find the current character, but a simpler way of getting the current character is destructuring.
[Destructuring](https://clojure.org/guides/destructuring#_sequential_destructuring) made the overall code much simpler.
I can peek the starting characters, and also keep the original string.

```clj
[first-char second-char third-char :as source] "abcde"
=> \a \b \c "abcde"
```

To simplify the individual conditional cases, instead of manually keeping track of index, used functions which specialized in string / collection processing.

For e.g.

```
var_name = 10;
^       ^
start   end
```

The earlier implementation would start from the start index, keep increasing index till it finds whitespace.
Then use these two indexes to build the identifier. 
This is really error prone.

Instead of manually finding ending index, a simpler way is to just, 

```clj
(split-with letter? "var_name = 10;")
=> 
[(\v \a \r \_ \n \a \m \e) (\space \= \space \1 \0 \;)]
```

pass the entire input string to a function which is responsible for matching characters given a predicate.
The first part is then converted to a token, and `lex` called recursively on rest of the string.
The final result is first token conj'd to result of the recursive call.

## Parser

The list of tokens carries no semantic meaning. 
The parsing phase is responsible for converting a list of tokens, to a tree which conforms to [C's grammar](https://www.lysator.liu.se/c/ANSI-C-grammar-y.html).

```clj
(def ex "
static int x = 20;

int main(void) {
  int y = 22;
  return x + y;
}
")

(parse-from-src ex)

=> 
[{:type :declaration,
  :declaration-type :variable,
  :variable-type {:type :int},
  :storage-class :static,
  :identifier "x",
  :initial
  {:type :exp, :exp-type :constant-exp, :value {:type :int, :value 20}}}
 {:type :declaration,
  :declaration-type :function,
  :function-type
  {:type :function, :return-type {:type :int}, :parameter-types []},
  :storage-class nil,
  :identifier "main",
  :parameters [],
  :body
  [{:type :declaration,
    :declaration-type :variable,
    :variable-type {:type :int},
    :storage-class nil,
    :identifier "y",
    :initial
    {:type :exp, :exp-type :constant-exp, :value {:type :int, :value 22}}}
   {:type :statement,
    :statement-type :return,
    :value
    {:type :exp,
     :exp-type :binary-exp,
     :binary-operator :plus,
     :children [:left :right],
     :left {:type :exp, :exp-type :variable-exp, :identifier "x"},
     :right {:type :exp, :exp-type :variable-exp, :identifier "y"}}}]}]

```

The book only implements a subset of C, so the parser does not handle all valid code. For e.g.
```c
int x = 20, y = 22; // not supported by the parser, even though it's valid
```

Each language construct is composed of smaller constructs. 
Program is a list of functions. Function is list of declarations or statements and so on.
This tree like data structure needs to be represented in code, and Clojure maps are the perfect to do it.

```clj
{:type :exp,
 :exp-type :binary-exp,
 :binary-operator :plus,
 :children [:left :right],
 :left {:type :exp, :exp-type :variable-exp, :identifier "x"},
 :right {:type :exp, :exp-type :variable-exp, :identifier "y"}}
```

Each node in this tree contains keys which help identify it's semantic meaning.
The above node represents a binary expression, and has keys for it's children nodes.

Maps are a joy to work with. 
Unlike in a typed language (talking about type system like Java, C++) where I would have had to create new classes for each construct in the C grammar, maps help avoid that boilerplate.
I don't have to create specialized `get, update` like functions for any nodes. 
As all the nodes are map, all Clojure functions which operate on maps work.

This was a double edged sword though. 
As the compiler cannot guarantee at compile time what keys are present and no autocomplete, it became harder to keep track of the keys present in a node. 

Anytime I would have to use something, I would need to go back to the constructor function and remember what name did I gave to those keys.

```clj
(defn binary-exp-node [l r op]
  {:type :exp
   :exp-type :binary-exp
   :binary-operator op
   :children [:left :right]
   :left l
   :right r})
```

This was tedious and error prone, but there is no way to solve this problem ( at least in terms of autocomplete ). 
I thought records could help, but apart from giving automatic constructor function, they did not help to solve this problem. 
It's still difficult to keep track of, and make sure of the keys present in any map.

To slightly alleviate this problem, I used a schema library.

### Using a Schema Library

There are mainly two benefits to using a schema.

- Central place which defines what keys / value can be in a map.
- Perform runtime validation 

I first tried [clojure.spec](https://clojure.org/guides/spec), and it did work but I felt as I added more constructs it became too complicated, at least for my simple use case.
I just wanted to specify the keys on my map, and what value they should be.

I went with [metosin/malli](https://github.com/metosin/malli). 
It's way of specifying schema was exactly what I was looking for.

```clj
(def BinaryExp
  [:map
   [:type [:= :exp]]
   [:exp-type [:= :binary-exp]]
   [:binary-operator `[:enum ~@(set (keys t/bin-ops))]]
   [:left [:ref #'Exp]]
   [:right [:ref #'Exp]]
   [:children [:= [:left :right]]]
   [:value-type {:optional true} #'Type]])
```

Above is the schema for a binary expression.
All the keys are there, and what values they can be. 

I used malli's validation at the end to verify whether my program's output conforms to the schema.

```clj
(m/coerce #'s/Program program) ; Validates whether my program conforms to the program schema
```

This function is present at the boundary of compiler pass.
Before passing the result to the next function, I validate whether the AST created is valid and conforms to the schema.
As the name of the keys are now centralized and with runtime validation, I caught bugs much earlier and was able to add more constructs easily, as the next compiler pass always had valid AST as input.

### Parsing 

#### Instaparse

Instead of trying to write my own parser, I  used [Instaparse](https://github.com/Engelberg/instaparse).
And it worked great. 
I already had the grammar with me, and was able to use Instaparse to quickly write a parser.
It's a feature rich library and was great, until I had to implement operators with precedence.
It's possible to write a grammar which handles precedence, but it's not fun.
The grammar ends up containing intermediate rules, and as C has a lot of operators this was getting out of hand.
I ended up hand writing the parser.

#### Handwritten 

The parser has some primitive functions.
These functions are then combined in bigger functions, which handle individual rules in the grammar. 
My implementation closely resembles the one mentioned in the book.

```clj
(defn- expect
  "Expects the first token in list to be of given kind.

  Returns the token and remaining tokens."
  [kind [token & rst]]
  (if (= kind (:kind token))
    [token rst]
    (exc/parser-error "Actual and expected token differ.")))
    
    
(defn- parse-repeatedly
  "Repeatedly runs given parse function on input until end-kind encountered.

  `parse-f` must return result in form [node remaining-tokens]."
  [tokens parse-f end-kind]
  (loop [res []
         tokens tokens]
    (if (= end-kind (:kind (first tokens)))
      [res tokens]
      (let [[node rst] (parse-f tokens)]
        (recur (conj res node) rst)))))
        
```

These functions are called inside main language constructs.

```clj
(defn- parse-if-statement [tokens]
  (let [[_ tokens] (expect :kw-if tokens)
        [_ tokens] (expect :left-paren tokens)
        [exp-node tokens] (parse-exp tokens)
        [_ tokens] (expect :right-paren tokens)
        [then-stmt tokens] (parse-statement tokens)
        else? (= :kw-else (:kind (first tokens)))]
    (if (not else?)
      [(if-statement-node exp-node then-stmt) tokens]
      (let [[_ tokens] (expect :kw-else tokens)
            [else-stmt tokens] (parse-statement tokens)]
        [(if-statement-node exp-node then-stmt else-stmt) tokens]))))
        
(defn- parse-block [tokens]
  (let [[_ tokens] (expect :left-curly tokens)
        [block-items tokens] (parse-repeatedly tokens parse-block-item :right-curly)
        [_ tokens] (expect :right-curly tokens)]
    [block-items tokens]))
```

The general structure of each function is expecting certain tokens, such as parenthesis, curly braces etc.
If there is a list of things, then use `parse-repeatedly`.
Store intermediate AST nodes in the `let` blocks. 
At the end, return the AST node, and the remaining tokens.

I don't like how this looks, but it works.
It's super simple to add more constructs. 
( At least it was for the current implemented grammar. Maybe after pointers, arrays this will become difficult to maintain). 
I wanted to implement this in a different way.

I tried to first convert this into some sort of threading macro.
```clj
(defn parse-if-statement [tokens]
  (->> tokens
       (expect :kw-if)
       (expect :left-paren)))
```

The next step expects an expression, but I need to hold onto that ast node of the expression.
So I would have needed to store that in a let block anyway, or thread it through the remaining functions, which didn't look that simple to me.
I wasn't really able to figure out a better way to write the parser. 
Once the general structure was there, it was easy to add new constructs and I kept going.

Below are some resources I found which implement a parser in Clojure.
I haven't been able to give them enough time, but just on a high level overview their implementation looks to be more declarative.
I will refactor the parser in future.

- [Parsing C using Clojure by David Hovemeyer](https://daveho.github.io/2017/01/05/parsing-c-using-clojure.html)
- [Writing Parser Combinator Library in Clojure by Dmitry Geurkov
](https://troydm.github.io/blog/2016/04/11/writing-parser-combinator-library-in-clojure/)

---

One thing which in general irked me throughout was whether I am doing things `clojure way`. 
(Especially with how I implemented the parser.)
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

There are a lot of such instances, especially in the next pass of the compiler, where I want to refactor.
There seems to be a lot of repetitive code, but it's mixed with the complexity of C specification as well (for e.g. storage class specifiers), so I still haven't been able to refactor it to something I am happy with.

One of the solutions to this problem ( or in general ), I guess would be reading more code.
I have watched a lot of conference talks, but throughout building this I realized I haven't actually read more codebases.
One of the projects which helped me and gave new ideas is [tools.analyzer](https://github.com/clojure/tools.analyzer). I will describe how it helped in the [tacky](#tacky) section.

Listed below are some articles, projects which can help to find other projects, ideas etc for implementing things in `clojure way`.

- [Which Clojure codebases should I read? How and why? by Aditya Athalye](https://www.evalapply.org/posts/which-clojure-codebases-to-read-how-and-why/index.html#main)
- [A Clojure view of "Mars Rover" by Aditya Athalye](https://www.evalapply.org/posts/clojure-mars-rover/index.html#main) Models the mar's rover problem keeping Clojure's it's just data ideology
- [Timothy Baldridge - Data All The ASTs](https://youtu.be/KhRQmT22SSg?si=Mu-Jw-1fEm12Od6N) Clojure way of processing ASTs
- [Clojure Design Patterns by Mykhailo Kozik](https://mishadoff.com/blog/clojure-design-patterns/)

---

## Analyze

The analyzer phase consists of three smaller passes, but their core is the same.
All of them work with the parsed AST from previous step, and go through these steps:

1. Resolve: Validates things such as variables re-declared twice in same scope.
Variables used before declared, nested function definitions etc.
    
2. Label Loop: Loops, break, continue statements require labels to identify them.
This ends up being in the final assembly to create labels for jumps statements.

3. Typecheck: Typechecks variable declaration, function calls etc. 
Also adds type information to each AST node.

The main addition in this phase is of the symbol map.
Each identifier in the program is now tracked.
It's name, storage class, type, initial value etc. 
How should this map ( which is required for the above validation ) be passed around in different function calls.

For e.g.

```c
Map<String, Attributes> m = new Map();

handle_variable_declaration(ast, m) {
  string var_name = ...;
  Attribute attr = ...;
  m.put(var_name, attr);
} 

// similarly for functions, compound statements etc

typecheck_program(ast, m) {
  handle_variable_declaration(ast, m);
  handle_function_declaration(ast, m);
}
```

There is a map, which we access by reference. 
This map contains the type information and other attributes.
This reference is passed around to all the functions.

In imperative language this is simple. 
Things such as hashmap are by references, and it can be updated from any function, nested to any levels and it still works.

I wasn't sure how to port this to Clojure.
As maps are immutable, even if I `assoc/update` new values onto a map, it won't reflect outside.

```clj
(defn handle-var-decl [ast m]
  (...
  (assoc m :key :value))) ; this change not reflected in the m passed as parameter
```

We can return an updated map from this function. 
Then the next function which requires the map for it's processing, would need to require it.
The problem was how to glue these functions together. 
How to easily pass the updated map from function to another ?
The solution was using `reduce`.

```clj
(defn typecheck-declaration [ast m]
  (...)
  {:declaration (...)
   :ident->symbol (updated-m m)})

(defn- typecheck-program [program]
  (let [rf (fn [acc decl]
             (let [d (typecheck-declaration decl (:ident->symbol acc))]
               {:program (conj (:program acc) (:declaration d))
                :ident->symbol (:ident->symbol d)}))]
    (reduce rf
            {:program []
             :ident->symbol {:at-top-level true}}
            program)))
```

Let's say we process a declaration. 
This declaration adds a new variable to the map, with it's type, initial value etc.
After we process the declaration, we return the updated symbol map and set it in the accumulator.
This map is then passed to the next declaration, as it's available in the reducer's accumulator.

Same approach for compound statements, which are list of statements.
Each statement processing function, returns the updated map, which is used in the next statement call.

A easier way to do this is using `atom`, but I wanted to first try solving this without relying on atoms. 
I did use atoms later on in a different phase, but for now returning an updated map from every function worked fine.

Overall this phase of the compiler returns the same AST as previous phase, but with extra type information and the symbol map.

```clj
;; same input program as above
{:program
 [{:type :declaration,
   :declaration-type :variable,
   :variable-type {:type :int},
   :storage-class :static,
   :identifier "x",
   :initial
   {:type :exp, :exp-type :constant-exp, :value {:type :int, :value 20}}}
  {:type :declaration,
   :declaration-type :function,
   :function-type
   {:type :function, :return-type {:type :int}, :parameter-types []},
   :storage-class nil,
   :identifier "main",
   :parameters [],
   :body
   [{:type :declaration,
     :declaration-type :variable,
     :variable-type {:type :int},
     :storage-class nil,
     :identifier "y.0",
     :initial
     {:type :exp,
      :exp-type :cast-exp,
      :target-type {:type :int},
      :typed-inner
      {:type :exp,
       :exp-type :constant-exp,
       :value {:type :int, :value 22},
       :value-type {:type :int}},
      :children [:value],
      :value
      {:type :exp,
       :exp-type :constant-exp,
       :value {:type :int, :value 22},
       :value-type {:type :int}},
      :value-type {:type :int}}}
    {:type :statement,
     :statement-type :return,
     :value
     {:type :exp,
      :exp-type :binary-exp,
      :binary-operator :plus,
      :children [:left :right],
      :left
      {:type :exp,
       :exp-type :variable-exp,
       :identifier "x",
       :value-type {:type :int}},
      :right
      {:type :exp,
       :exp-type :variable-exp,
       :identifier "y.0",
       :value-type {:type :int}},
      :value-type {:type :int}}}]}],
 :ident->symbol
 {"x"
  {:type {:type :int},
   :attribute
   {:type :static,
    :initial-value {:type :initial, :static-init {:type :int-init, :value 20}},
    :global? false}},
  "main"
  {:type {:type :function, :return-type {:type :int}, :parameter-types []},
   :attribute {:type :fun, :defined? true, :global? true}},
  "y.0" {:type {:type :int}, :attribute {:type :local}}}}
```

## Tacky

The book introduces an intermediate IR representation called Tacky.
Expressions from above passes are converted to a [three-address code](https://www.geeksforgeeks.org/three-address-code-compiler/).
Below is an example output from this pass. 
The instructions key now has instructions such `jump`, `copy`, `add` etc.
This is closer to the assembly representation.
This pass reduces the nested expressions to variables.
The operands to each instruction are either a constant value, or a variable.

```clj
(def source 
"int foo(void) {
  int x = 0;
  for (; x < 21; x += 1)
    ;
  return x;
}

int main(void) { return foo() + foo(); }")

(tacky-from-src ex)
=> 
{:program
 [{:type :declaration,
   :declaration-type :function,
   :function-type
   {:type :function, :return-type {:type :int}, :parameter-types []},
   :storage-class nil,
   :identifier "foo",
   :parameters [],
   :global? true,
   :instructions
   [{:type :copy,
     :src {:type :constant, :value {:type :int, :value 0}},
     :dst {:type :variable, :value "x.196"}}
    {:type :label, :identifier "for_start.198"}
    {:type :binary,
     :binary-operator :less-than,
     :src1 {:type :variable, :value "x.196"},
     :src2 {:type :constant, :value {:type :int, :value 21}},
     :dst {:type :variable, :value "binary_result_less_than.200"}}
    {:type :jump-if-zero,
     :identifier "break_for_label.197",
     :val {:type :variable, :value "binary_result_less_than.200"}}
    {:type :label, :identifier "continue_for_label.197"}
    {:type :binary,
     :binary-operator :add,
     :src1 {:type :variable, :value "x.196"},
     :src2 {:type :constant, :value {:type :int, :value 1}},
     :dst {:type :variable, :value "binary_result_add.199"}}
    {:type :copy,
     :src {:type :variable, :value "binary_result_add.199"},
     :dst {:type :variable, :value "x.196"}}
    {:type :jump, :identifier "for_start.198"}
    {:type :label, :identifier "break_for_label.197"}
    {:type :return, :val {:type :variable, :value "x.196"}}
    {:type :return, :val {:type :constant, :value {:type :int, :value 0}}}]}
  {:type :declaration,
   :declaration-type :function,
   :function-type
   {:type :function, :return-type {:type :int}, :parameter-types []},
   :storage-class nil,
   :identifier "main",
   :parameters [],
   :global? true,
   :instructions
   [{:type :fun-call,
     :identifier "foo",
     :arguments [],
     :dst {:type :variable, :value "function_call_result_foo.201"}}
    {:type :fun-call,
     :identifier "foo",
     :arguments [],
     :dst {:type :variable, :value "function_call_result_foo.202"}}
    {:type :binary,
     :binary-operator :add,
     :src1 {:type :variable, :value "function_call_result_foo.201"},
     :src2 {:type :variable, :value "function_call_result_foo.202"},
     :dst {:type :variable, :value "binary_result_add.203"}}
    {:type :return, :val {:type :variable, :value "binary_result_add.203"}}
    {:type :return, :val {:type :constant, :value {:type :int, :value 0}}}]}],
 :ident->symbol
 {"foo"
  {:type {:type :function, :return-type {:type :int}, :parameter-types []},
   :attribute {:type :fun, :defined? true, :global? true}},
  "x.196" {:type {:type :int}, :attribute {:type :local}},
  "main"
  {:type {:type :function, :return-type {:type :int}, :parameter-types []},
   :attribute {:type :fun, :defined? true, :global? true}},
  "binary_result_add.199" {:type {:type :int}, :attribute {:type :local}},
  "binary_result_less_than.200"
  {:type {:type :int}, :attribute {:type :local}},
  "function_call_result_foo.201"
  {:type {:type :int}, :attribute {:type :local}},
  "function_call_result_foo.202"
  {:type {:type :int}, :attribute {:type :local}},
  "binary_result_add.203" {:type {:type :int}, :attribute {:type :local}}}}

```

Each construct in the previous AST node, is converted to instructions in three address code format. 

```clj
(defn if-statement-handler [s symbols]
  (let [cond-exp (run-expression-handler (:condition s) symbols)
        cond-value (:val cond-exp)
        cond-instructions (:instructions cond-exp)
        then-instructions (statement->tacky-instruction (:then-statement s) symbols)
        end-label (label "if_end")
        else-label (label "if_else")
        else? (:else-statement s)]
    (if else?
      [cond-instructions
       (jump-if-zero-instruction cond-value else-label)
       then-instructions
       (jump-instruction end-label)
       (label-instruction else-label)
       (statement->tacky-instruction (:else-statement s) symbols)
       (label-instruction end-label)]
      [cond-instructions
       (jump-if-zero-instruction cond-value end-label)
       then-instructions
       (label-instruction end-label)])))
```

Above is converting `if statements` to instructions in TAC form.
It first recursively generates the instructions for expressions inside a statement.
In above it's the condition expression. 
The other statements inside the `if` block are generated, and the result from all these instructions is returned as a list of instructions.

The main change is now using `symbols` parameter. Which is an atom, containing a map from variable identifiers to their type / initial value etc.
Always returning an updated version of this map felt tedious now, and for this phase I used an atom.

The other place which felt repetitive was how I processing the AST nodes.
Below is the schema for an expression.

```clj
(def VariableExp
  [:map
   [:type [:= :exp]]
   [:exp-type [:= :variable-exp]]
   [:identifier string?]])
   
(def UnaryExp
  [:map
   [:type [:= :exp]]
   [:exp-type [:= :unary-exp]]
   [:unary-operator `[:enum ~@t/unary-ops]]
   [:value [:ref #'Exp]]
   [:children [:= [:value]]]])

(def BinaryExp
  [:map
   [:type [:= :exp]]
   [:exp-type [:= :binary-exp]]
   [:binary-operator `[:enum ~@(set (keys t/bin-ops))]]
   [:left [:ref #'Exp]]
   [:right [:ref #'Exp]]
   [:children [:= [:left :right]]]])

(def Exp
  [:schema {:registry {::mexp-variable #'VariableExp
                       ::mexp-unary #'UnaryExp
                       ::mexp-binary #'BinaryExp}}
   [:multi {:dispatch :exp-type}
    [:variable-exp #'VariableExp]
    [:unary-exp #'UnaryExp]
    [:binary-exp #'BinaryExp]]])

```

`Exp` is by it's nature recursive. 
A binary expression operands can be any other expression, such as functional call, unary expression etc.

To process expressions, I started with the above approach.

```clj

(defmulti exp-handler [e]
  (:type e))
  
(defmethod exp-handler :binary
  [exp symbols]
  (let [{v1 :val
         insts1 :instructions} (exp-handler (:left exp))
        {v2 :val
         insts2 :instructions} (exp-handler (:right exp))
        op (binary-operator (:binary-operator exp))
        dst (variable (str "binary_result_" op))
        _ (add-var-to-symbol dst (tc/get-type exp) symbols)
        binary-inst (binary-instruction op v1 v2 dst)]
    {:val dst
     :instructions (flatten [insts1
                             insts2
                             binary-inst])}))

```

Each handler, internally called `exp-handler` again.
This felt just extra work, as it's a recursive definition there should be a way to recursively process this map.
When I access the `:left` on the binary expression, it should already contain the evaluated value.

I looked into how to operate on recursive tree like data structures in Clojure.
I first looked into [clojure.zip](https://clojuredocs.org/clojure.zip).
This blog post [Clojure Zippers by Ivan Grishaev](https://grishaev.me/en/clojure-zippers/) and talk [The Art of Tree Shaping with Zippers by Arne Brasseur](https://youtu.be/5Nm56YvTKZY?feature=shared) are a great intro.
But the conclusion I came up with was that this seemed to be doing too much, and I definitely do not get the full potential of using zippers.
I just wanted to have pre computed value in my maps.

The second thing was [clojure.walk](https://clojuredocs.org/clojure.walk/walk). 
This blog post by [Learning to walk with Clojure by Abhinav Omprakash](https://www.abhinavomprakash.com/posts/clojure-walk/) explains the walk functions in details with a lot of examples.

The function `postwalk` seemed to be the solution.
It does post order traversal, calling a supplied `f` on child nodes.
It replaces each child node with `(f child)`.
This was exactly what I was looking for.

```clj
(clojure.walk/postwalk
 (fn [x]
   (prn x)
   (if (vector? x)
     x
     (* x 2)))
 [1 [2] [3 4 [5]]])
 
=> 
1
2
[4]
3
4
5
[10]
[6 8 [10]]
[2 [4] [6 8 [10]]]
```

The final result is returned at the last, after all the children node are processed.
I can use the same approach for recursively running the expression handlers, but how to identify the children ? 
The above example is a nested vector, and each element is valid node. But in the AST map, only specific keys are recursive in nature.

The solution to this was specifying children nodes in the AST itself.
Then use this to write a custom postwalk function. 
This solution was suggested by [hiredman on Clojurians Slack](https://github.com/hiredman).
The same approach was first used in [tools.analyzer](https://github.com/clojure/tools.analyzer).
Its described in these talks [Timothy Baldridge - Data All The ASTs
](https://youtu.be/KhRQmT22SSg?feature=shared) and [Clojure eXChange 2015 Immutable code analysis with tools analyzer](https://youtu.be/oZyt93lmF5s?feature=shared).

The custom postwalk function is below. It know's what are the children nodes from the ast itself, and then recursively processes them first.

```clj
(defn postwalk [ast f]
  (f (reduce
      (fn [acc key]
        (let [value (get acc key)]
          (if (vector? value)
            (assoc acc key (doall (map (fn [node] (postwalk node f))
                                       value)))
            (assoc acc key (postwalk value f)))))
      ast
      (:children ast))))
```

After this, the code for handling expressions is now simpler.

```clj
(defn binary-exp-handler
  [exp symbols]
  (let [{v1 :val
         insts1 :instructions} (:left exp) ; not running the expression-handler again
        {v2 :val
         insts2 :instructions} (:right exp)
        op (binary-operator (:binary-operator exp))
        dst (variable (str "binary_result_" op))
        _ (add-var-to-symbol dst (tc/get-type exp) symbols)
        binary-inst (binary-instruction op v1 v2 dst)]
    {:val dst
     :instructions (flatten [insts1
                             insts2
                             binary-inst])}))
```

## Assembly

This pass has the most amount of code. 
It converts the Tacky AST to Assembly instructions.
There's a lot of logic and edge cases to be handled, so that the assembly produced is compliant.

- Functions have to follow [System V calling](https://en.wikipedia.org/wiki/X86_calling_conventions). 
First 6 parameters are passed in registers, other on the stack. 
- Stack pointer needs to be a multiple of 16. As different types take up different space, this needs to maintained manually by rounding to nearing offsets.
- Several assembly instructions cannot take both operands as memory addresses, so they are rewritten using intermediate registers. 

There are other smaller details such as above, but apart from that this phase codewise is not doing anything interesting.
It's again dispatch on type of the instruction from the previous Tacky phase, and generating assembly instructions.

```clj
;; generates ret instruction in the assembly output
(defmethod tacky-instruction->assembly-instructions :return
  [{return-value :val} m]
  (let [src (tacky-val->assembly-operand return-value) ; return value
        reg (reg-operand :ax) ; which operand to in mov instruction
        src-type (tacky-val->assembly-type return-value m)]
    [(mov-instruction src-type src reg) (ret-instruction)])) ; moves value into eax, returns
```

Assembly instructions take operands, but they can only take a specified combination of operands.
Some cannot take in immediate values in the source, some cannot take both operands to be from the stack. 
In some instructions if the specified constant value is too large ( out of 32 bit integer range ), then temporary registers needs to be used. 
I initially started with using `cond` blocks to handle all these if conditions, but it was getting out of hand. I ended up using [core.match](https://github.com/clojure/core.match).

These talks were a great introduction to the library [Sean Johnson - Pattern Matching in Clojure](https://www.youtube.com/watch?v=n7aE6k8o_BU) and [Understanding pattern matching in Clojure with core.match
 by Misophistful](https://www.youtube.com/watch?v=mi3OtBc73-k). Example of using match, with maps and having guard conditions.

```clj
(defn- fix-cmp-instruction [instruction]
  (let [src (:src instruction)
        dst (:dst instruction)
        assembly-type (:assembly-type instruction)
        imm-outside-range? (fn [o] (and
                                    (= :imm (:operand o))
                                    (not (util/in-int-range? (:value o)))))]
    (match [instruction]
      [{:src {:operand (:or :data :stack)} ; operand can be either on stack or data section
        :dst {:operand (:or :data :stack)}}] [(mov-instruction assembly-type src (reg-operand :r10))
                                              (cmp-instruction assembly-type (reg-operand :r10) dst)]
      [({:assembly-type :quadword
         :src {:operand :imm}
         :dst {:operand :imm}} ; guard on operand being in integer range
        :guard [(comp imm-outside-range? :src)])] [(mov-instruction :quadword src (reg-operand :r10))
                                                   (mov-instruction :quadword dst (reg-operand :r11))
                                                   (cmp-instruction :quadword (reg-operand :r10) (reg-operand :r11))]
      [({:assembly-type :quadword
         :src {:operand :imm}}
        :guard [(comp imm-outside-range? :src)])] [(mov-instruction :quadword src (reg-operand :r10))
                                                   (cmp-instruction :quadword (reg-operand :r10) dst)]
      [{:dst {:operand :imm}}] [(mov-instruction assembly-type dst (reg-operand :r11))
                                (cmp-instruction assembly-type src (reg-operand :r11))]
      :else instruction)))
```

Below is an output from the assembly phase.

```clj
(def input "int foo(int x) {
  if (x == 42) {
    return x;
  }
  return foo(x + 1);
}

int main(void) { return foo(0); }")

(asembly-from-src input)
=> 
{:program
 [{:op :function,
   :identifier "foo",
   :global? true,
   :instructions
   [{:op :binary,
     :binary-operator :sub,
     :assembly-type :quadword,
     :src {:operand :imm, :value 16},
     :dst {:operand :reg, :register :sp}}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :reg, :register :di},
     :dst {:operand :stack, :value -4}}
    {:op :cmp,
     :assembly-type :longword,
     :src {:operand :imm, :value 42},
     :dst {:operand :stack, :value -4}}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :imm, :value 0},
     :dst {:operand :stack, :value -8}}
    {:op :setcc, :operand {:operand :stack, :value -8}, :cond-code :e}
    {:op :cmp,
     :assembly-type :longword,
     :src {:operand :imm, :value 0},
     :dst {:operand :stack, :value -8}}
    {:op :jmpcc, :identifier "if_end.234", :cond-code :e}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :stack, :value -4},
     :dst {:operand :reg, :register :ax}}
    {:op :ret}
    {:op :label, :identifier "if_end.234"}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :stack, :value -4},
     :dst {:operand :reg, :register :r10}}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :reg, :register :r10},
     :dst {:operand :stack, :value -12}}
    {:op :binary,
     :binary-operator :add,
     :assembly-type :longword,
     :src {:operand :imm, :value 1},
     :dst {:operand :stack, :value -12}}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :stack, :value -12},
     :dst {:operand :reg, :register :di}}
    {:op :call, :identifier "foo"}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :reg, :register :ax},
     :dst {:operand :stack, :value -16}}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :stack, :value -16},
     :dst {:operand :reg, :register :ax}}
    {:op :ret}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :imm, :value 0},
     :dst {:operand :reg, :register :ax}}
    {:op :ret}]}
  {:op :function,
   :identifier "main",
   :global? true,
   :instructions
   [{:op :binary,
     :binary-operator :sub,
     :assembly-type :quadword,
     :src {:operand :imm, :value 16},
     :dst {:operand :reg, :register :sp}}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :imm, :value 0},
     :dst {:operand :reg, :register :di}}
    {:op :call, :identifier "foo"}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :reg, :register :ax},
     :dst {:operand :stack, :value -4}}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :stack, :value -4},
     :dst {:operand :reg, :register :ax}}
    {:op :ret}
    {:op :mov,
     :assembly-type :longword,
     :src {:operand :imm, :value 0},
     :dst {:operand :reg, :register :ax}}
    {:op :ret}]}],
 :backend-symbol-table
 {"foo" {:type :fun-entry, :defined? true},
  "x.232" {:type :obj-entry, :static? false, :assembly-type :longword},
  "main" {:type :fun-entry, :defined? true},
  "binary_result_equal.233"
  {:type :obj-entry, :static? false, :assembly-type :longword},
  "binary_result_add.236"
  {:type :obj-entry, :static? false, :assembly-type :longword},
  "function_call_result_foo.237"
  {:type :obj-entry, :static? false, :assembly-type :longword},
  "function_call_result_foo.238"
  {:type :obj-entry, :static? false, :assembly-type :longword}}}
```

## Emission

This is simplest phase of the compiler.
It just takes in the assembly in ast form, and converts it to assembly instructions. 
E.g.

```clj
(defn- mov-instruction-emit [instruction]
  (let [atype (:assembly-type instruction)
        opts {:register-width (assembly-type->operand-size atype)}
        src (operand-emit (:src instruction) opts)
        dst (operand-emit (:dst instruction) opts)
        suffix (assembly-type->instruction-suffix atype)]
    [(format "    %s%s        %s, %s" "mov" suffix src dst)]))
```

The generated assembly is slightly different, whether the system is Linux or Mac, but otherwise there isn't much going on in these functions, just string formatting.

Below is an example of final assembly generated.

```clj
(def input 
"int main(void) {
  static int x = 1;
  if (x == 42)
    return x;
  x = x + 1;
  return main();
}")

(emit input)
```

```assembly
    .data
    .balign 4
x.251:
    .long 1


    .globl main
    .text
main:
    pushq       %rbp
    movq        %rsp, %rbp
    subq        $16, %rsp
    cmpl        $42, x.251(%rip)
    movl        $0, -4(%rbp)
    sete        -4(%rbp)
    cmpl        $0, -4(%rbp)
    je         .Lif_end.253
    movl        x.251(%rip), %eax
    movq        %rbp, %rsp
    popq        %rbp
    ret
    .Lif_end.253:
    movl        x.251(%rip), %r10d
    movl        %r10d, -8(%rbp)
    addl        $1, -8(%rbp)
    movl        -8(%rbp), %r10d
    movl        %r10d, x.251(%rip)
    call        main
    movl        %eax, -12(%rbp)
    movl        -12(%rbp), %eax
    movq        %rbp, %rsp
    popq        %rbp
    ret
    movl        $0, %eax
    movq        %rbp, %rsp
    popq        %rbp
    ret


.section .note.GNU-stack,"",@progbits
```

---

This was my first largeish (for me) project in Clojure, or Lisp in general.
I got familiar with a lot of libraries, talks and patterns.
I tried different editor setups throughout (Vim + Conjure, Vscode + Calva). 
Currently using Doomemacs.

Using the REPL for immediate feedback was probably my favorite part in all this. 
Trying out my solution without the overhead of writing it in a test, or generating executable and then running it helped me iterate on solutions quicker and led me to experiment more than I probably would have.
Using [flowstorm](https://github.com/flow-storm/flow-storm-debugger) for debugging is amazing.
It has saved me a lot of time, which would have been spent debugging by printing the values etc.

I am yet to complete the book. Many of the upcoming features are core to C, such as pointers, arrays, structs.
As of now, I feel the base structure of the compiler is there, and each feature can be neatly added on top.
But as those features are more complex than the ones presented above, there is a still a gradual difficulty curve,
and I notice more opportunities to make my code simpler.
Looking forward to implementing the rest of the book !
