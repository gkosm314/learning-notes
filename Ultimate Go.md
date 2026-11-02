---

---
TODO:
- read about Golang strings
- read about struct padding (generally)
	- examples
	- why float64 is 8-aligned (SIMD, cache line split)
	- understand when we add padding at the end (why is it enough to take care of the largest element)
- mess around with pointers in Golang
# 1. Introduction 
- Productivity vs Performance
	- writting code fast vs writing fast code
	- Golang aims to maintain productivity while allowing you to write programs that are fast enough.
	- Golang aims to provide a clear mental model over how the program will be executed
- Correctness vs Performance
	- correct > clear > concise > fast, in that order
	- there is no perfect solution, just trade-offs we have to decide about
### Software Engineering tips
- Be tolerant when you see bad code, you don't know the circumstances/constraints the engineer was working under.
- It is important to maintain a mental model of the code you are working with.
	- Don't depend too much on the debugger to the point that your mental model of the program becomes weak. (you will not have a debugger in production!)
### Writing code
- Integrity is about being reliable.
	- micro level: memory writes are correct, type systems help with this
	- macro level: data transformations are correct, writing less code and handling errors helps with this
	- when writing code, focus on failure, not on the happy path
- Readability is not hiding complexity, it is about making things easy to understand.
- Simplicity is hiding complexity, but without losing readability.
	- achieved through refactoring, not from day one
- First make something work, then make it reliable, then make it readable, then make it simple (hide **some** of the complexity).
## Performance from a high level
1. external latencies (system calls, network)
2. internal latencies (GC, synchronization)
3. memory accesses
4. algorithmic efficiencies
# 2. Memory & Data Semantics
## Variables
- type = size + what this piece of data means (= how to interpret the bytes)
- Go has zero-value = when we allocate memory and we don't initialize it, the compiler will initialize it with the zero-value
	- `var` does a zero-value allocation
- For numerical types, Go provides generic and precision types (`int`, `int32`, `int64`)
	- if you don't have a particular reason then just use `int` and let the compiler allocate the most efficient type
- Strings in Go are two-word allocations
	- first word will be a pointer (zero-value is nil)
	- second word will be the number of bytes (zero-value is 0)
	- when the string is initialized, a string will be allocated, the pointer points to the beginning of the allocation and the length will be equal to the bytes used by the string
- Conversion vs casting
	- assume we have a type that allocates one byte and interprets it as an integer
	- casting: read the byte and the next 3 bytes and interpret them as a 4-byte integer
		- still possible in Go using the `unsafe` package
	- conversion: allocate 4 bytes and copy the 1 byte to the new allocation
		- costs in performance but it is safer
		- default in Go
## Struct  Types
- Structs allow you to define your own types
### Padding 
- Initially you would think that the struct would occupy exactly as much memory as the sum of its field. Wrong!
- The hardware can only read/write at the **word level**. Words have specific "starting points". For example in a 32-bit machine, the word is 4 bytes longs, and a word can start from byte 0, byte 4, byte 8 etc.
- If a field's memory allocation belongs to two words, then reading it would take two read operations, since we cannot read the last half of one word and the first half of the next word. This is slow.
- In order to ensure that this will not happen, the compiler will add "padding" bytes after a field to make sure that the next field will not belong to two words (it will be "aligned"). If needed, it will also add padding bytes at the end of the struct, to ensure that if we have arrays of this struct, the fields of each struct will belong to exactly one word.
	- there are rules for all that
- We can improve the padding situation by padding the fields by largest to smallest.
### Syntax
```
// define a named type
type example struct{
	counter int16
	flag    bool
}

// both e1 and e2 are zero-value initializations, they do the same thing
var e1 example
e2 := example{}

// initialization
e3 := example{
	counter: 3,
	flag: true
}

// zero-value initialization with literal type 
var e4 struct{
	counter int16
	flag    bool
}

// initialization with literal type
e5 := struct{
	counter int16
	flag    bool
}{
	counter: 3,
	flag: true
}

type bill struct{
	counter int16
	flag    bool
}

type nancy struct{
	counter int16
	flag    bool
}

var b bill
var n nancy
b = n // this will fail! Golang is strongly typed and it does not allow implicit conversion.
b = bill(n) // in Golang we need explicit conversion.
// above example protects us from assigning an int to a uint and getting bugs from the implicit conversion

// the example below will not fail, because e5 uses a literal type, not a named type
b = e5
```
## Pointers
- When a Go program starts, the Go runtime looks to see how many logical processors (cores) we have.
	- physical processors * physical cores/processor * hardware threads/physical core = hardware thread
	- hardware thread = what we use to run OS thread = how many goroutines we can run concurrently
- The Go runtime creates
	- P = logical processor
	- M = machine = live operating system thread
	- M scheduled by OS to run to be run by a hardware thread
- Every Goroutine gets its own contiguous block of memory, called a stack. OS threads also get their own stack.
	- Goroutines get very small stack (2K) in order to have a small overhead that will allow us to have many many goroutines if needed.
- Every time a function is called, a frame for that function is created in the stack. The function only operates inside that frame (sandboxed).
	- a way to think about it: functions get input and produce output, everything they need lives inside their frame (= sandbox)
- When you use a variable, you get its value.  **By using the operator `&` you get the address of the variable.**
- **When passing a variable to a function, you pass by value** => we make a copy of the variable in the function's frame and the function will operate on that copy.
	- this is safe because it does not allow to modify in-place (no side effects)
	- but not allowing in-place modification is also a restriction (what if i want to pass the value to N functions and have each one of them updating it)
	- it is also inefficient because we have to copy the data
- We can use the `*` operator in the function signature to pass the address of the variable.
	- inside the function we can use `*` to modify **the value of** the variable that the pointer is pointing to
	- this allows the function to modify memory outside of its frame, we say that the goroutine has shared access
	- we cannot use plain `*` in the signature, compiler needs to know the type because we will modify the variable
- If we return a pointer to a value we created, it escapes up to a frame above us. In that case, this value is created in the heap.
	- we still get to write code using the value semantics
	- we just return the heap address to our caller
	- TL;DR: compiler decides if the variable will be allocated to heap (there is also a flag that says us why it decided that)
	- readability guidelines: always use value construction, unless we are returning in place. goal is to make it obvious when we share the address and to use value semantics everywhere else.
- ^ The cost of this is that the garbage collector has to get involved!