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

each instruction is broken up into 3 stages (Fetch, Decode, Execute). At every clock cycle, we complete 1 stage of an instruction.

Because of the multi-cycle design, shorter instructions can also take less cycles to complete. For example, a MULT instruction may take more instructions than an ADD instruction because of the more complex arithmetic logic in the MULT instruction.

Note:
- At any given time, **only one stage of one instruction** is active.
    
- Instruction 2 starts **only after** Instruction 1 finishes.
    
- Therefore, the CPU is **not pipelined** — it cannot fetch the next instruction while executing the current one.

**Pipelined Processors:** The core idea of pipelining is to overlap the execution of multiple instructions. The need of pipelined processors come from the fact that executing 1 instruction at a time severely limits the throughput of the processor.

The analogy:
![[Pasted image 20251015141522.png]]

Pipelining is a highly effective technique that significantly improves efficiency by allowing multiple instructions to be processed simultaneously at different stages of execution

However, branch instructions in particular introduce complications in pipelining because of how these types of instructions alter the flow of instruction execution.

## Branch Prediction

- **Purpose:**  To predict whether a **branch instruction** (e.g., `if`, `goto`, `loop`) will be **Taken** or **Not Taken** early in the pipeline (usually at the **Fetch stage**).

- **Goal:**  Reduce stalls and wasted clock cycles by pre-loading the correct instructions before knowing the actual branch result.

- **Execution stage confirmation:**  The processor only knows for sure whether the branch is taken or not in the **Execute (EXE)** stage.

- **Pipeline flush penalty:**
	- If prediction is **wrong** → pipeline must be **flushed** (instructions discarded) → CPU stalls and waits.
	- If prediction is **correct** → pipeline continues smoothly → significant **time saved**.

### **Types of Branch Prediction**

1. **Static Branch Prediction**
	- Decision is based **only on the instruction itself**.
	- **No history** of previous branch outcomes is used.
	- Example: always predict “Not Taken” or use simple heuristics (e.g., backward branches are taken for loops).
2. **Dynamic Branch Prediction**
	- Uses **historical information** (past Taken/Not Taken patterns).
	- Adapts to the program’s **runtime behavior**.
	- More accurate than static methods.

### Branching in Database Systems
The Filter Example
In a **DBMS sequential scan**, we often execute code like:
```
for (int i = 0; i < n; i++) {
    if (column[i] < 100) {  // filter condition (branch)
        // process the tuple
    }
}
```

Here, every row checked triggers a **branch** — whether the tuple passes the predicate or not.
 
Why This Branch Is “Nearly Impossible to Predict Correctly”
Branch predictors work best when patterns are **regular**:
- Example: always true, always false, or alternating predictably.
But in a database filter, the condition result (`column[i] < 100`) is **data-dependent**:
- Each row may pass or fail **randomly**, depending on data distribution.
- For example, if ~50% of rows match, the branch outcome flips frequently (True/False/True/False…).

From the CPU’s point of view → it looks _random_.  
Therefore, **branch prediction accuracy drops drastically** (often below 60%).
When prediction fails:
- The CPU must flush the pipeline on every misprediction.
- This causes **high stall time**, reducing throughput dramatically.