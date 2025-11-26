## Variable Initialization (Initial-ization)

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