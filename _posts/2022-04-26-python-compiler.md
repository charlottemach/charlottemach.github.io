---
layout: post
title: "Writing a compiler in Python"
date: 2022-04-26
tags: python compiler
---

After not quite finishing writing a compiler in SML in uni I've decided to give it a second shot in Python. 
This is based on the amazing Andrew W. Appel's [compiler book](https://www.cs.princeton.edu/~appel/modern/java/),
writing a compiler for the Tiger language in Python.

## General

While a compiler can be easily summarized as a tool that's "translating programming languages into executable code",
the different phases of it are a bit more complex.

Below is the breakdown similar to the summary in Appel's book:
* Lex: Breaking the code into pieces (so called tokens)
* Parse: Analyzing the structure
* Semantic Actions: Creating an abstract syntax tree for each piece
* Semantic Analysis: Understanding the meaning of each piece
* Frame Layout: Adapt to machine
* Translate: Create intermediate representation trees
* Canonicalize: Clean up
* Instruction Selection: Grouping parts of the trees to machine instructions
* Control Flow Analysis: Create graph from instructions
* Dataflow Analysis: Understand what happens to variables during execution
* Register Allocation: Choose right registers for variables
* Code Emission: Use registers with machine instructions

## Lex

Similar to what the book suggests, we can use libraries to make our life easier. From JavaCC and SableCC to [PLY](http://www.dabeaz.com/ply/ply.html).
We add a file defining what are allowed tokens, reserved words, what constitutes comments, what's allowed for ids and so on.
I refered back to [this Tiger language reference manual](https://www.lrde.epita.fr/~tiger/tiger.html#Lexical-Specifications) for a few things.

## Parse

Using PLY we can also write a grammar file, based on the [EBNF here](https://assignments.lrde.epita.fr/reference_manual/tiger_language_reference_manual/syntactic_specifications/syntactic_specifications.html).
Things to look out for in Pyton:
* In theory you can write a function for each syntax specification, but it makes sense to group them for later use
* It helps to use a custom data structure (like tree nodes) to keep the parsed data
* That also makes the output/visual verification much easier (e.g. using pydot)
* Debugging is simplest with a custom format (see Python [logging module](https://docs.python.org/3/library/logging.html))
