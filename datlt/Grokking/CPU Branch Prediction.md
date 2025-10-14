lowest level executes instructions according to a clock. 
Fundamental types of instructions:

1. Data Movement Instructions: instructions that transfer data between memory `LD` (load data from memory to a register), `ST` (store data from register to memory), `MOV` (data from register to register)
2. Arithmetic Instructions: instructions that do quick maths with that data `ADD``(add 2 register values together), `SUB` (subtracts 2 register values together), `MUL`(multiplies 2 regiser values together), `DIV` (divide a register value by the other)
3. Control flow instructions: instructions that direct the CPU to execute instructions in a different order. `JMP`, `BEQ` (branch). These instructions are usually translated from conditional statements such as `if` or `while` conditionals.

High Level Language -> Assembly code -> Machine code 