---
title: C++ Compilation Introduction
date: 2025-09-30
categories: [C++, Compilers]
tags: [c++, compilers]
---

# C++ Compilation Introduction

## General Overview

### Preprocesing Phase

This phase transforms the raw source file (.cpp) into an expanded translation unit (.i file) and is done before syntax parsing. When the compiler sees #include <> or #include ".h", it recursively replaces the include header with the entire content of that file or library. This can resilt in dozens or hundreds of files getting pulled into a single gigantic .cpp file. This is why it's so important to not include unnecessary headers repetitively.  This is also what guards like #pragma once or #ifnder, #define, and #endif are used for; preventing duplicate inclusion into this massive .cpp file.

### Macro Expansion

The preprocessor performs macro substitution, very similarly to how constexpr will be evaluated later on and replaces the variable usage with the value it is defined to. IE: #define X 1, float n = X + X -> float n = 1 + 1.

### Conditional Compilation

Using #if, #ifdef, #ifndef, #else, #elif, #endif, platform or config specific code can be conditionally include or excluded. For example, you will probably need different behaviors for different operating systems (depending on situation of course) and can instruct it which sections will be used and which to compile.

### Line Control

One really cool thing that the C++ compilers do is use #line row_number "originalFileName.cpp". This is because when everything gets dumped into one condensed file, the traceback should an error arise would be completely scrambled and impossible to find. This way, it can tell you on exactly which line in which file something when wrong. There are a lot of these if you inspect the .i file.

### End Result

At the end, there is a translation unit (intermediate .i file) that has flattened the source code into one file, resolved all includes, macros, conditionally excluded code, and line locations. This is what gets passed to the compiler itself.

## Compilation Phase (Frontend)

The compiler receives this translation unit and now the code needs to be validated.

### Lexical Analysis

Here, a language parser turns the characters in a line into tokens that signify an action. For example, given: int output = a + b * 2;, it will convert this to something like [TOK_INT] [TOK_IDENTIFIER: output] [TOK_EQUALS] [TOK_IDENTIFIER: a] [TOK_PLUS] [TOK_IDENTIFIER: b] [TOK_MUL] [TOK_INT_LITERAL: 2] [EOS] (not actual just example for understanding). Now, the compiler has actions that it can map. Additionally, it removes comments and escape characters, etc here.

### Syntax Analysis

After the lexical analysis is done, the output is taken and organized into an Abstract Syntax Tree (AST) which is a structured representation of the code's syntax. It uses grammars to build valid trees and this is where syntax errors like a forgotten ; or unmatched parenthesis are caught. An example of what the tree might look like for a simple chunk of code is: for(int i = 0; i < 5; ++i) { n += i; }. The resulting AST might look something like:
ForLoop:
- Initialization: VarDeclaration(int, i, 0)
- Condition: LessThan(i, 5)
- Change: Increment(1)
- Body:
    - Expression Statement: Assignment(n, +=, i)

### Semantic Analysis

Now, the compiler pivots from purely semantics and grammer to actual intent.

#### Symbol Resolution

It resolves every identifier to its declaration and checks if variables are in scope, were they declared before use, are the correct namespaces and class-level scope applied, etc. 

#### Type Checking and Overload Resolution

The compiler now makes sure that expressions match expected type (meaning datatypes are operation compatible, so adding an int and a double is, but adding an int to a string is not), function calls are made with the correct arg types and overloads, and implicit conversions for nonhomogenous operations are allowed. The templates aren't instantiated just yet, but it will still validate template declarations to some degree.

#### Constant Folding and Evaluation

This is where the compile time expressions are calculated such as constexpr expression and if constexpr branches. It may even throw in some optimizations by eleminating branches whose end result is known. Optimizations like this are one of the reasons that C++ is blistering fast during execution, but painfully slow to compile compared to other languages.

#### Type Deduction

Figures out what auto means wherever used. I'd be very interested to see how this is done for more complex expressions, but generally I would assume this is done by tracing the datatypes of the elements in the operations and then recursively calculating the outputs of the results until the type is deduced. Also, I know that in Python, the declaration of type is implicit, obviously, but is almost explicit with the type of parenthesis used for certain datastructures, so I wonder if that is an aspect as well.

## Intermediate Code Generation

This is the in-between step designed to make optimization easier. ASTs focus on code structure, but IR focuses on operations and flow. These are typically expressed in static single assignment form where each variable is assigned exactly once.

### IR Optimizations

This is where the big gains in performance and code size happens. Some examples of these optimizations are:
- Loop unrolling: Replicating the loop body to reduce branch overhead (Can also be signaled explicitly with #pragma unroll)
- Replacing expensive operations with cheaper ones (ie instead of multiplying by 2, just push <<1)
- Invariant hoisting: move calculations outside the loop if they don't change.
- Replacing known values with their actual results (if known).
- Dead code elimination (unused variables).
- Function inlining: A function call has overhead because it has to be looked up on the function table and added to the stack, etc so it is significantly cheaper to simply copy and paste that function's contents into wherever it was called. This way its a win-win: Devs get the cleanness of separating code into functions and the performance as if it were still there.

## Optimizations in Depth

### Levels

Using flags, the aggressiveness of the optimizations can be set. These include:
- -O0: No optimizations for fast compile and easier debugging.
- -O1: Very basic like removing deadcode and inlining.
- -O2: Loop unrolling and scheduling.
- -O3: Very aggressive, enables vectorization (pulling in data at the L1 cache level and registers), inlining heuristics, etc.
- -Ofast: Very fast but risky that operations may be a little off.
- -Os/-Oz: Optimize for memory instead of speed.

Compilers are ridiculously advanced at optimizing. They're so good, that often, the execution speed bad code and mediocre code is similar. It's so fascinating to see just how far they go. Inlining is great, but things like vectorization where they can do out-of-order execution in order to utlize the data already existing in a cache (and hence access is significantly faster), all with the program still working as expected is absolutely incredible. Pair that with the fused operation chips coming out and the results are disgustingly good.

## Backend Code Generation

This is where the optimized IR gets translated into assembly for the target architecture. This is typically done using Directed Acyclic Graphs to match fragments. This is where things start to get really interesting. The order these instructions are executed in matters for performance and it also matters for correctness. Modern CPUs have multiple execution units of different types and can stall on cache misses or missing dependencies. Therefore, the instruction scheduler reorders instructions to hide latency (by inserting independent ops between loads and uses) and exploit parallelism as much as possible. Additionally, in the IR, the variable names were made unique, but the concept uses virtual registers which is obviously not the physical case. The backend maps these to a finite set of physical registers using a few different methods:
- Graph Coloring: Builds an inference graph where each node is a variable and edges connect variables that are live at the same time, then assigns registers such that no connected nodes share a register.
- Linear Scan: Simpler alternative used by JIT (Just in time) compilers that traverse live range and allocate using a greedy algorithm.
This is extremely important because if the registers fill up, something else has to be evicted and choosing the wrong values to evict repeatedly causes a massive increase in load time because it has to be pulled up from lower and lower caches.

The output is now an assembly .s file. From here, the assembler turns this into machine code. This is why when you download an executable from code built in C or Cpp, you don't need to install any additional things and also why you need a specific version: Your device already knows how to run it, but only a very specific version is compatible with your device.