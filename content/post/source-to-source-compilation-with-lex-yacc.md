---
title: "Source-to-source compilation with Lex-Yacc"
date: 2017-07-15T10:43:23-06:00
comments: true
tags: ["Python", "Lex/Yacc", "Compilers"]
categories: ["Development", "Projects"]
---
In this post, I will describe a source-to-source compiler that transforms a Brainfuck program into its equivalent 64 bits assembly code. The assembly program can be built into a executable, thus allowing you to run Brainfuck programs natively.
<!--more-->

A source-to-source compiler is a compiler that takes a program written in a certain language and outputs the equivalent program in another language. This process is useful to compile a program to an intermediate language, or to provide backward compatibility for a legacy language. The Babel compiler, for instance, can parse Javascript ES6 to Javascript ES5 standard. The ES6 standard comes with a lot of improvements and new features, but it is currently not supported by most web browsers. Or maybe, you can come up with a new language and translate it to another one. Take for example this list of [languages that compile to Javascript](https://github.com/jashkenas/coffeescript/wiki/list-of-languages-that-compile-to-js).


<a class=button href="https://github.com/eariassoto/brasm/" target="%5fblank">Get the code on GitHub</a>

# Requirements
I used Python to develop the compiler. In order to run the compiler you need to have Python installed on your computer. Also, you need the Python Lex-Yacc (PLY) module installed in your python environment. You can download the latest PLY version (I used the 3.10 version) from the [PLY homepage](http://www.dabeaz.com/ply/). To install PLY uncompress the file you downloaded, open a terminal within the uncompressed folder and run:

```bash
$ sudo python setup.py install
```

In case you want to compile the assembly code, you need the [x86asm assembler](http://www.x86asm.us/), and the Linux dynamic linker (ld command). I decided to go with x86asm because it supports the Intel assembly syntax which is more readable than the AT&T syntax.


# The source language: Brainfuck

Brainfuck is a  [Turing complete](https://en.wikipedia.org/wiki/Turing_completeness) programming language. I chose this language as input language because it has a small set of instructions. Its only memory structure is a buffer of at least 30,000 bytes initialized to zero. In C, that buffer can be declared as `char buffer[30000] = {0};`. The only variable is a data pointer that starts at the beginning of the buffer. To continue the C analogy, the data pointer might be declared as `char *ptr = buffer;`

A Brainfuck program is structured as a series of instructions. The most basic instructions manipulate memory buffer using the data pointer. There are two streams of bytes for input and output. Also, there are two instructions to implement loops. Table below describes all the instructions:

| Command | Description | C equivalent |
| ------------- |:-------------:| -----:|
| > | Increment data pointer | ++ptr; |
| < | Decrement data pointer | --ptr; |
| + | Increment by one byte at the data pointer | ++*ptr; |
| - | Decrement by one byte at the data pointer | --*ptr; |
| . | Print byte at the data pointer | putchar(*ptr); |
| , | Read char from input, store in the byte at the data pointer| *ptr=getchar(); |
| [ | If the byte at the data pointer is zero, jump to matching `]` | while (*ptr) { |
| ] | if the byte at the data pointer is nonzero, jump to matching `[` | } |

Brainfuck falls into the category of [esoteric programming languages](https://en.wikipedia.org/wiki/Esoteric_programming_language). So, Brainfuck programs are really hard to understand. For example, this is an implementation of the classic "Hello World" program written in Brainfuck:
```brainfuck
++++++++++[>+++++++>++++++++++>+++>+<<<<-]
>++.>+.+++++++..+++.>++.<<+++++++++++++++.>.+++.------.--------.>+.>.
```

# Hello Assembly
For this post, I will assume you are familiarized with x86-64 assembly code. To compare the previous example, I wrote a Hello World in x86-64 assembly code.

```GAS
global _start

section .text
_start:
mov rax, 1
mov rdi, 1
mov rsi, message
mov rdx, 13
syscall

mov eax, 60
xor rdi, rdi
syscall

message:
db "Hello, World", 10
```

The `global _start` directive defines the entry point for our program. The `.text` section contains the program instructions and all constants we defined. This program calls two special functions, called system calls. System calls are functions executed by the kernel because they required higher privileges. These functions take parameters using registers, and they are invoked using the `syscall` instruction. This “hello world” program calls the  `sys_write` system call to print the string `"Hello World"`, and the `sys_exit` system call to exit the program. To check all the available system calls and their parameters, [check this list](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/).

# Translating Brainfuck instructions set to x64 assembly
The first thing I needed to define was the buffer memory and the data pointer. I declared the buffer as a 30000 bytes zero-initialized array in the `.bss` section. In this section, the program declared static variables. These variables will be allocated and initialized by the program loader. All variables in the `.bss` section are zero-initialized. For the data pointer, I will use register `r9`. When the program starts, the buffer address will be stored in this register.

```GAS
section .bss
buffer: resb 30000

_start:
mov r9, buffer
```

I will break Brainfuck instructions into four subsections: pointer instructions, data instructions, input/output instructions, and the loop instruction.

## Pointer instructions
Instructions `>` and `<` move the data pointer through the buffer. For this task, address in register `r9` should move eight positions (so the address moves a byte) forward or backwards.

```GAS
; > move data pointer forward
add r9, 8

; < move data pointer backward
sub r9, 8
```

## Data instructions
Here, the byte stored in the address in register `r9` is modified using a temporary register, `r10b`. Then, the result value is stored back into the same address.
```GAS
; + add 1 to byte at data pointer
mov byte r10b, [r9]
add r10b, 1
mov byte [r9], r10b

; - subtract 1 to byte at data pointer
mov byte r10b, [r9]
sub r10b, 1
mov byte [r9], r10b
```

## Input/Output instructions
To provide the input and output data streams, I used system calls. These are functions provided and executed by the kernel that allow a program to interact with higher level functionality. Before calling a system call the register `rax` must have the system call id number. Some system calls may take parameters from specific registers. The compiler uses the `sys_write` and `sys_read` system calls to read and write from `stdin` and `stdout` respectively.

The `_print_byte` function prints an input byte on `stdout`. Input byte must be store on the register `rcx`. The function pushes the register `rcx` into the stack. Then, it prepares the registers needed by the system call `sys_write`. Register `rdi` contains the file descriptor. In this case `stdout` file descriptor is `1`. Register `rsi` has the address of the content to be written, in our case the stack address. Register `rdx` has the length of the buffer. Finally, register `rax` contains the system call id, for `sys_write` the id is `1`. Before calling `_print_byte` function, the program will store on register `cl` (lowest byte in register `rcx`) the byte at data pointer.

```GAS
_print_byte:
push rcx ; grow stack and push output byte to stack
mov rdi, 1 ; file descriptor 1 is stdout
mov rsi, rsp ; stack address
mov rdx, 1 ; number of bytes to print
mov rax, 1 ; system call 1 is write
syscall ; call OS system call
pop rcx ; decrease stack
ret

; . call _print_byte function
mov cl, [r9]
call _print_byte
```

The `_read_byte` function is very similar to the `_print_byte` function. The only difference is that `sys_read` id number is `0`, and the file descriptor for `stdin` is also `0`. After calling `_read_byte` function, the program stores the byte returned by `_read_byte` on `cl` at data pointer.


```GAS
_read_byte:
sub rsp, 8 ; grow stack
mov rsi, rsp ; stack address
mov rdi, 0 ; file descriptor 0 is stdin
mov rdx, 1 ; number of bytes
mov rax, 0 ; syscall 0 is read
syscall ; call OS system call
pop rcx ; decrease stack and save input
ret

; , call _read_byte function
call _read_byte
mov byte [r9], cl
```

## Loop instruction
The instructions `[` and `]` are the C equivalent of `while (*ptr) { }`. The assembly loop jumps into the comparison between the byte at data pointer and zero. If the value is not zero, the program jumps to the `_cloop` tag. Between the `_cloop` and `_loop` tags, the compiler will put all the parsed Brainfuck code that was originally between `[` and `]` instructions.
```GAS
jmp _loop0
_cloop0:
; [loop code]
_loop0:
mov byte cl, [r9]
cmp cl, 0
jne _cloop0
```
# Building the compiler
Compiling a program can be described, in a very basic way, as a two parts process:

1. Break the input program into the language components.
2. Build the program using the language components and the language specification.

This is where Lex-Yacc comes to use. Lex will help us parsing the input program from a series of characters to strings with certain meaning, called tokens. This process is known as lexical analysis. Yacc will generate a parser from a grammar. The parser will take the tokens and try to find a valid match for the grammar. In this process, the grammar rules will trigger actions that will create the output code.

## Lexical analysis
Here, I will go into details about what the PLY Lex implementation needs to build a lexical analyzer. First thing, Lex requires a list of token names. I defined one token for each Brainfuck instruction.

```python
tokens = (
    'INCR_PTR', 'DECR_PTR', 'INCR_VAL', 'DECR_VAL', 'WRITE', 'READ', 'LB', 'RB'
)
```
Each token needs a rule to match the input with the token. Rules are defined as functions named as the token with the prefix `t_`. Some rules are simple, and can be defined as regular expressions. I defined simple regular expressions for each Brainfuck instruction.

```python
# Simple Tokens
t_INCR_PTR = r'\>'
t_DECR_PTR = r'\<'
t_INCR_VAL = r'\+'
t_DECR_VAL = r'\-'
t_WRITE = r'\.'
t_READ = r'\,'
t_LB = r'\['
t_RB = r'\]'

t_ignore = " \t" # Ignored characters
```
Some rules need to perform an action for every token they find. For example, I will set a rule to count the number of lines in order to indicate the line number in a compilation error. Also, I will treat any other characters the input program might have as comments.

```python
def t_newline(t):
    r'\n+'
    t.lexer.lineno += t.value.count("\n")

def t_error(t):
    """ Ignore all other characters """
    t.lexer.skip(1)

```
With these, we can tell Lex to build the lexical analyzer.
```python
# Build the lexer
import ply.lex as lex
lex.lex()
```

## Parsing the tokens
Yacc will generate our parser based on a [Backus–Naur form (BNF) grammar](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form). The grammar I used is simple because Brainfuck programs are just a sequence of valid instructions. I defined two types of instructions: simple instructions (`> < + - . ,`) and loops (`[ ]`). Loops are treated as a different type of instructions because Brainfuck allows nested loops. The entire grammar is specified below.

```
program : instruction

instruction : simpleInstruction
| loopInstruction
| simpleInstruction instruction
| loopInstruction instruction

loopInstruction : LB instruction RB

simpleInstruction : INCR_PTR
| DECR_PTR
| INCR_VAL
| DECR_VAL
| WRITE
| READ
```

Grammar rules are defined in Yacc as functions. These functions need the prefix `p_` and take one parameter named `p`. `p` is an array containing the values of each grammar symbol in the corresponding rule. The rule is written in a triple quote comment after the function definition. The function body will be the actions that the rule will perform if it founds a match. Rules that are reduced should save in `p[0]` the final result.

The first rule defined will be the starting point for the parse. These is the starting rule for my compiler:
```python
def p_program(p):
    ''' program : instructions '''
    global global_success
    compile_program(p[1])
    global_success = True
```
This rule will translate all the Brainfuck instructions into their assembly equivalents. The `compile_program` function will replace Brainfuck operations into assembly instructions and save the result on a file. The next rules reduce all the instructions into the final `instructions` symbol.

```python
def p_instructions_expansion(p):
    ''' instructions : simpleInstruction instructions
    | loop_instruction instructions
    '''
    p[0] = p[1] + p[2]

def p_instructions(p):
    ''' instructions : simpleInstruction
    | loop_instruction
    '''
    p[0] = p[1]
```
The rules to match the instructions will reduce the instruction within a wrapper structure. The wrapper will store the instruction type and the instruction (or instructions in case it is a loop).

```python
def make_instruction_action(type, payload):
    """ Wraps the instruction content into a struct """
    obj = {}
    obj[INSTR_OBJ_PROP_TYPE] = type
    obj[INSTR_OBJ_PROP_PAYLOAD] = payload
    return obj

def p_loop_instruction(p):
    """ loop_instruction : LB instructions RB
    """
    action = make_instruction_action('loop', p[2])
    p[0] = [action]


def p_simple_instruction(p):
    """simpleInstruction : INCR_PTR
    | DECR_PTR
    | INCR_VAL
    | DECR_VAL
    | WRITE
    | READ
    """
    action = make_instruction_action('op', p[1])
    p[0] = [action]
```

I also defined a rule in case the parser encounters a error in the input program.

```python
def p_error(p):
    if p:
        print("Syntax error at '%s'" % p.value)
    else:
        print("Error, input program empty.")
```
Finally, I just need to build the parser and give the compiler a starting point.

```python
import ply.yacc as yacc
yacc.yacc()
```

```python
def main():
    global global_compiled_filename
    parser = argparse.ArgumentParser()
    parser.add_argument("file", help="Brainfuck input file to be parsed")
    parser.add_argument("-o", help="file name for the output assembly code")
    args = parser.parse_args()
    if args.o:
        global_compiled_filename = args.o
    try:
        f = open(args.file, 'r')
        program_str = f.read()
        f.close()
        result = yacc.parse(program_str)

    if(global_success):
        print("Program compiled. To build the executable run:\
        \n\tx86asm -f elf64 {0}.asm && ld {0}.o -o {0}".format(global_compiled_filename))
    except IOError:
        sys.exit('Error, input file not found!')
```

# Improvement proposals
* Optimize the compiler. Group pointer and data instructions to no repeat a lot of assembly code.
* Support reading/writing files. Perhaps the memory buffer can be loaded from/saved into a file.
* Extend Brainfuck specification to support variables and string constants.
## Bigger challenge
* Create a compiler to translate a subset of the C language to Brainfuck. Then, use this compiler to translate the Brainfuck code into assembly.

<a class=button href="https://github.com/eariassoto/brasm/" target="_blank">Get the code on GitHub</a>
