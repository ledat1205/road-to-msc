# C++ Variable Assignment & Initialization Cheat Sheet

This cheat sheet summarizes the key concepts for defining, assigning, and initializing variables in C++.

---

## 1. Variable Assignment

**Assignment** is the process of giving a variable a new value _after_ it has already been defined.

|**Concept**|**Syntax**|**Description**|
|---|---|---|
|**Assignment Operator**|`=`|Used to copy the value on the right-hand side (RHS) into the variable on the left-hand side (LHS). This overwrites any previous value.|
|**Example**|`int width;`<br><br>  <br><br>`width = 5;`|First, define the variable.<br><br>  <br><br>Then, assign a value.|

> ⚠️ **Common Warning**: Do not confuse the **Assignment Operator** (`=`) with the **Equality Operator** (`==`), which is used to test if two values are equal.

---

## 2. Variable Initialization (Initial-ization)

**Initialization** is the process of specifying an **initial value** for a variable at the moment it is defined, combining two steps into one.

### The 5 Common Forms of Initialization

The C++ standard includes several forms for initialization. **List-initialization** is the modern, preferred approach.

|**Form**|**Syntax/Example**|**Description**|**Modern?**|
|---|---|---|---|
|**Default-initialization**|`int a;`|**No initializer provided.** Often leaves the variable with an **indeterminate ("garbage") value** (unless it's a class type or global variable). **Avoid.**|No|
|**Copy-initialization**|`int b = 5;`|Initial value provided after the **`=`** sign. Inherited from the C language.|Legacy|
|**Direct-initialization**|`int c(6);`|Initial value provided in **parentheses `()`**.|Legacy|
|**Direct-list-initialization**|`int d {7};`|Initial value in **braces `{}`**. This is a form of **List-Initialization** and is generally **preferred**.|**Yes**|
|**Value-initialization**|`int e {};`|**Empty braces `{}`**. In most cases for fundamental types, this performs **Zero-initialization** (sets the value to 0).|**Yes**|

### Key Benefit of List-Initialization (`{}`)

List-initialization (`{value}` or `{}`) is preferred because it **disallows narrowing conversions**.

|**Conversion Type**|**Syntax**|**Result**|**Status**|
|---|---|---|---|
|**Narrowing Conversion**|`int w { 4.5 };`|Compiler **Error** (Prevents data loss of `.5`).|**Safe**|
|**Non-List Conversion**|`int w = 4.5;`|Compiles, but `w` is silently initialized to `4`.|Risky|