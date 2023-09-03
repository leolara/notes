
# A software Engineer perspective on the future of ZKP [DRAFT]

I have been a software engineer in different advanced industries for more than 20 years, and recently I am happy to have done a deep change to the world of Zero Knowledge proofs. I am surrounded by other types of professionals that have been previously in my career, mostly mathematically and cryptography researchers. This is very exciting for me, as I never want to stop learning, and it is also not the first time to have jumped to a new area and find myself with people of other backgrounds, like when switching to scientific computing or in the realm of CPU development.

In this context, I find myself asking this ludicrous question:

How is going to be the PHP of ZKP?

ZKP is a space dominated by cryptographers, mathematicians and many people with a PhD. The contrast is huge with the language that runs Wordpress, and that undermost of the world content marketing. But precisely because of that it is a good starting point to break the barriers that ZKP currently have for mass adoption, its blindspots, and fulfil its potential as a game changer when a infinite garden of composable ZKP applications are created.

One of the concepts in software engineering that we are missing in ZKP is the stack, from the opcodes in your CPU to the website you are reading right now, there is a huge stack of different technologies each one empowering the next one.

Let's adventure on the current situation of ZKP stack and helping us with the tool of finding similarities with the computing stack.
## Proving systems

Probably 95% of the development in ZKP currently it is being done here. This is the consequence that most people working on this space are researchers on the fields of mathematics and cryptography. This is also a reflection on what early stage this space is, we are just seeing the baby steps of ZKP into the world.

Every week, there is one or several papers with new ideas and new implementations of these ideas. It is frenetic, if we see this as comparable to the computing world to CPUs development, it is likely that its improvements will become exponential if they are not already.

This must make us hopeful that in the future we will be able to create ZKP circuits for projects that seem unreachable right now in terms of proving system capacity.
## Arithmetization

Most current ZKP researchers stop here and it is call front-end by them, this is perfectly reasonable, it the same way a CPU architect calls front-end to the fetch and decode cycles of the processor.

It works as an abstraction for the proving systems, in a very similar way that machine code is an abstraction to program a CPU. At the beginning of computation, people working with computers had to write programs by hand in machine code. However, now it is obvious that the simplest program we work with today would be highly impractical to program this way.

## Low level DSL

With computers we have assembly language that is just a simple and readable way of writing machine code. The next step was macro assembly, that adds a layer to avoid to copy/paste code many times. As powerful as this macro systems could get, you still could mechanically by hand translate a macro assembly program to machine code if you have enough time.

It is happening exactly the same in the ZKP space, creating DSLs and APIs that are direct translation of arithmetizations with tools to avoid code repetition . A clear example is circom and RC1S. Circom constraints can be directly translated to R1CS, with circom templates helping with code repetition.

Unfortunately for most of my colleges in the ZKP space it is difficult to see beyond this. It is understandable, their mental space is the proving systems, and for them arithmetizations are the frontend. This leads to a lack of research on the possibility on adding more layers to the stack beyond this, with the  only exception of zk-VMs.

Zero Knowledge proof virtual machines have become a huge part of ZKP specially due to the important benefit of being able to prove blocks on turing complete blockchains like for example zkEVM. And the logical next conclusion is instead of writing circuits to prove computation let's write a zkVM and compile the computation to its bytecode. This still ignores the fact that writing zkVMs directly on a arithmetization is a huge and unforgiving engineering task.

I know this from my own experience working on the zkEVM at EF, the original idea of this project and some others was to write a zkEVM directly on halo2 arithmetization. This has proof impossible to do, and it became only possible when a lot of ad-hoc software engineering abstractions were built on top of the arithmetization. With complexity abstraction becomes unavoidable.
## High level ZKP languages

Let's start by defining what are we talking about. We usually identify a programming language with a syntax and a file extension, but that is not what they are in a more abstract sense.

**A high level ZKP language is a set of abstractions that are close to express the intention of the ZKP developer and that need to be compiled non-trivially to an arithmetization in order to produce a ZKP circuit.**

So, what are there abstractions?

We don't know yet, and that is what makes this space, personally, the most exciting.

However, there is one that is obvious and that is **arbitrary complex boolean assertions**.

So, for example in a programming language like C:

```c
int generic_boolean_expr(int a, int b, int c) {
  return a && (b || !c);
}
```

This compiled to the following assembly:

```assembly
generic_boolean_expr(int, int, int):
	push rbp
	mov rbp, rsp
	mov DWORD PTR [rbp-4], edi
	mov DWORD PTR [rbp-8], esi
	mov DWORD PTR [rbp-12], edx
	cmp DWORD PTR [rbp-4], 0
	je .L2
	cmp DWORD PTR [rbp-8], 0
	jne .L3
	cmp DWORD PTR [rbp-12], 0
	jne .L2
.L3:
	mov eax, 1
	jmp .L4
.L2:
	mov eax, 0
.L4:
	movzx eax, al
	pop rbp
	ret
```

Nowadays, nobody would expect to write this boolean expression by hand in assembly.

We should put together this with the understanding that most ZKP developers will think about the circuit as boolean assertions about the computation being proven.

Hence we should not expect the ZKP developer to convert by hand these boolean assertions into R1CS polynomial constraints. I think this is the most obvious abstraction.

But this is not enough, in the same way we have functions, objects and classes, for loops et cetera in programming languages we need much more new abstractions on ZKP. This is when the future is less clear and also makes this a much more interesting area of research and development.

In the work at the zkEVM on top of halo2 arithmetization, many abstractions have come up. One I personally bet is the computational trace made of steps. In the end we need to proof the trace of a computation in every ZKP circuit, and that trace should be divided in atomic steps.

Other area of interest that is being actively research is typing of signals and expressions, this could help greatly with building sound circuits with less bugs and security issues.

A ZKP language will be parsed into a Abstract Syntax Tree or perhaps more aptly a Circuit Description Tree, and there will be a non-trivial compilation to the arithmetization. This opens the door to another advantage of high level ZKP languages. That the same ZKP circuit could be compiled to many arithmetizations and potentially in very different proving systems.

In conclusion, this is all new land that we are opening, a new frontier.

## Frontends for languages

You can observe that I have not talked about a particular syntax for the high level language, I think it is good to divide in this context between the language and the frontend to the language.

There are two main approaches that we can take here:
+ A custom parser and lexer.
+ Using a DSL on top of a existing programming language.

The first option is more work, and also can lead you in the path of having to create many custom tools like package managers and build systems without a clear advantage.

Using a existing language give you a mature tooling and a friendly ecosystem. Also some languages have good support to build readable DSLs on top of them. In my personal opinion this is a good option.

We can make a comparison with the Artificial Intelligence space, where no new from-scratch languages have find mass adoption, but languages on top of programming languages have flourished, like for example PyTorch.

There is another detail I would like to touch, for the second option, should we use typed or untyped languages as frontend? In most advanced programming typed languages are prefer because they give more robust software and also provide more structure to keep the code clean and following certain software engineering rules. However, these characteristics are not inherited by the ZKP circuit being written in a DSL in a type language, that doesn't make more robust or structured the circuit itself. Also, untyped languages are faster in terms of "keyboard typing" time, so it seems that probably an untyped language will be better for a ZKP DSL.

My personal favourite right now is Python, simply because it seems the most respected and widespread untyped language currently. Additional, we can see the success of AI and data science abstractions and frameworks with a Python frontend.

Note that this is not a reflection about in which language the proving system or even the language has to be written, just its frontend. Many advanced python libraries are written in C, C++ or Rust.
## zk-VMs

I would like to make an early distinction:
+ zk-VM can be a goal in itself, like for example a zkEVM, where we want to create proofs of a EVM based blockchain.
+ Seen as a tool to create ZK proofs creating a conventional computational program.

In this article we are interested about the second option. Here again we can distinguish about two options:
+ Based on a existing computer architecture like Risc-V or x86. (Computer Architecture)
+ Virtual machines with an architecture that are specific tailored to zero knowledge. (ZK Architecture)

Computer architecture zkVM has the advantage of leveraging existing tools and compilers. But, they are bringing abstractions that are very inefficient for zero knowledge proofs.

We could create ZK architecture VMs with different abstractions that the tradicional computer, and based on that specific programming languages that compile for such architecture.

One of the abstractions that we are bringing from computer architecture and are really inefficient in ZK is RAM. It is a pretty simple abstraction for computers, but in ZK we need proofs to allow the user to have random access memory.

We can envision a architecture without RAM. For example one with ROM for the bytecode and global constants, and all writable memory is in the local stack frame of the function currently being executed. As the stack frame is smaller than the whole RAM, proofs of access to it will as well be more smaller. To manage function calls and recursion, we could have a stack of stack frames which proof can be very simple by being the hash onion of the root of each stack frame. For complex data structure like arrays, we can custom proofs that follow the structure tree itself, and store and pass the root of each value.

Whatever the importance of zkVM is in general ZKP development in the future, these projects will be written in high level ZKP languages.

## Summary of ideas

These are my personal ideas from working in ZKP DSLs. I see a future where we have:
 + High level languages that are closer to the mental model of the ZKP developer and that can compile to different arithmetizations and proof systems.
 + Such languages will include arbitrary complex boolean assertions and other higher level abstractions like the computational step.
 + zk-VMs based on an architecture that break away from traditional computer architecture abstractions together with high level languages that match these new abstractions.

I think these tools will enable ZKP to really take up speed, attract much more people to work on these projects and enable the future we see ZKP is capable of.
