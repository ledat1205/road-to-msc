lowest level executes instructions according to a clock. 
Fundamental types of instructions:

1. Data Movement Instructions: instructions that transfer data between memory `LD` (load data from memory to a register), `ST` (store data from register to memory), `MOV` (data from register to register)
2. Arithmetic Instructions: instructions that do quick maths with that data `ADD``(add 2 register values together), `SUB` (subtracts 2 register values together), `MUL`(multiplies 2 regiser values together), `DIV` (divide a register value by the other)
3. Control flow instructions: instructions that direct the CPU to execute instructions in a different order. `JMP`, `BEQ` (branch). These instructions are usually translated from conditional statements such as `if` or `while` conditionals.

High Level Language -> Assembly code -> Machine code 

**Single Cycle Processors**
These are processors that execute 1 instruction per cycle, each cycle requires some constant amount of time, regardless of how complicated your instruction is

Slowest instruction has to be able to finish what its doing, meaning that the machine has to operate at the speed of the slowest instruction.

So they split the execution of an instruction to some small stages: 
1) Fetch 
2) Decode 
3) Execute 
4) Memory 
5) Writeback 
(The specific stages can vary depending on the design ).

All instructions in this phase are machine code (after compile)

`FETCH` : the instruction code gets loaded into the Instruction Register (IR), Program Counter (PC) will get incremented on the next cycle.

`DECODE`: the instruction code in the IR decodes the source and destination operands to determine the type of instruction it is.

`MEM`: If needed, data loaded onto the register is written to or read from memory.

`EXEC:` the Arithmetic Logic Unit (ALU) is passed appropriate control signals to complete arithmetic operations such as ADD, SUB, MULT etc. The ALU completes the required computation and loads the result to an intermediate register. In this stage, the processor usually resolves the branch condition and knows if a branch is taken or not.

`WRITEBACK:` any result computed by the instruction is written back to the register file (a place where we store all the values for our registers) to reflect the completion of the instruction.

**Multi Cycle Processors**
In a multi-cycle processor, each instruction is divided into several stages (e.g. Fetch, Decode, Execute, Memory, Writeback). Instead of the entire instruction being executed in 1 clock cycle, each stage is much shorter and is what determines the cycle duration

![[Pasted image 20251015110333.png]]


