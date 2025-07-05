## Memory Structure in C Programs

```
Memory Layout:
┌─────────────────┐
│   Stack         │ ←
│   (grows down)  │
├─────────────────┤
│   Unallocated   │
├─────────────────┤               ^
│   Heap          │ ←             |
│   (grows up)    │               |
├─────────────────┤      In order of lowest to
│   Data Segment  │ ←    highest memory address
├─────────────────┤               |
│   Read-Only     │ ←             |
│   Data Segment  │
├─────────────────┤
│   Code Segment  │ ←
───────────────────
```

### Segments

Each of the types of memory in the above diagram is called a segment.

#### Code Segment (AKA Text Segment)

Stores the executable compiled machine code. It is marked as read-only and executable by the OS. Attempting to write to the code segment will cause a segfault. This protects from code injection or accidental modification. Buffer overflow exploits, for example, will not work.

Multiple instances of the same program can share the code segment, saving memory, while each process would get its own copy of the other segments.

The compiler optimises the code before allocating to the code segment, including eliminating dead code (which doesn't get called).

#### Read-Only Segment

Stores data that should not be modified during execution. Read-only status is enforced at the hardware level by the OS and protects against accidental modification, which is particularly useful to avoid some types of security risks.

Global const and static const variables are allocated when the program starts and remain in memory until it terminates. They are also globally accessible.

**Examples**

```
// The following are stored in the read-only segment
const int MAX_SIZE = 100;     // Constants declared in the global scope
static const char* const VERSION = "1.0.0";         // Static constants
void (*functionPtr)(int) = someFunction;           // Function pointers
char* str1 = "Hello";                     // String literal - read-only
const int PRIME_NUMBERS[] = {2, 3, 5, 7};        // Arrays of constants

// Array copy (modifiable) - goes to stack/data segment
char str2[] = "Hello";
```

Attempting to modify this data results in a segfault.

Local const variables are stored on the stack, not the read-only data segment. They only exist for the duration of the function's lifetime and are still immutable, but this is enforced at the compiler level, not the hardware level.

#### Data Segment

Stores global and static variables, including local static variables. Uninitialised variables are initialised to `0` at runtime (or `NULL` for pointers).

Variables in the data segment persist for the entire lifetime of the program and, in the case of local static variables, they are not deallocated when the function returns.

**Example**

```
// These go in the Data Segment
int globalCounter = 0;
static char* globalString = "Hello";

void incrementCounter() {
    globalCounter++;  // Modifies data in Data Segment
}

int main() {
    static int localStatic = 5;  // Also in Data Segment
    incrementCounter();
    return 0;
}
```

Characteristics:

- Fast access
- Thread-safe for read operations
- Fixed size at compile time

#### Heap Segment

Used for dynamic memory allocation, which is where memory is requested from the heap at runtime. Memory is allocated with `malloc()`, `calloc()`, etc., and deallocated using `free()`. It is typically used for data that needs to persist beyond the lifespan of its local function.

Adding items to, and retrieving items from the heap entails slightly more overhead compared to the stack.

#### Stack Segment

The stack is commonly used for function call management and to store local variables during the execution of a function. When a function is called, a frame is added to the stack which contains local variables, function args and the return address, where execution should continue after the function returns.

The classic analogy of a stack of books works to a point. You can put things on the top of the stack and take things from the top, but that's where the similarities end, because you _can_ in fact read data from anywhere in the stack at any time.

This is possible because we also have stack pointers and frame pointers. The stack pointer points to the top of the stack and each frame has a frame pointer which usually points to the start of a given frame. When allocating memory, the compiler knows exactly how much space to use for each variable, so an offset from the frame pointer can be used to get any variable from any point the call stack.

```
                      Stack frames
                      ┌─────────────────┐
                      │ Previous        │
                      │ frame           │
                   ┌  ├=================┤
    │              │  │ Return addr.    │
 Direction         │  ├─────────────────┤
 of stack          │  │ Function args   │
  growth    Current│  ├─────────────────┤
    │        frame │  │ Frame pointer   │ <-- Frame pointer
    ⌄              │  ├─────────────────┤
                   │  │ Local vars      │
                   └  └─────────────────  <-- Stack pointer
```

Creating a thread starts a new call stack. Therefore, in order for their return values to be accessible in other areas of the code, they must store their values on the heap.

### Practical Applications

Now that we understand the uses for each segment, let's relate that to something more familiar and make the knowledge applicable.

**Stack Allocation:** Value types (`bool`, `int`, `char`, `long long`, etc.) are generally stored on the stack, but this depends on where and how they are declared.

Allocating to the stack.

```
void myFunction() {
    int x = 10; // 'x' is allocated on the stack
    printf("%d\n", x);
} // When myFunction returns, 'x' is popped off the stack and its memory is reclaimed.
  // You cannot access 'x' outside of myFunction.
```

**Heap Allocation:** Reference types (pointers) are references to variables that can be stored anywhere in memory. The pointer itself is typically stored on the stack, while the data it points to can be stored anywhere, but is usually allocated on the heap. This allows the data to be accessed across multiple functions because the pointer can be passed around and the data persists beyond the scope of the original function.

Memory allocated on the heap must be freed before the pointer is popped from the stack, because otherwise we won't have a way to reference the variable on the heap, so it can never be accessed or freed.

```
int* createAndReturnInteger() {
    // 'ptr' is a local pointer containing a memory address on the heap, stored on the stack
    int* ptr = (int*) malloc(sizeof(int));
    if (ptr == NULL) {
        // Handle allocation error
        return NULL;
    }
    *ptr = 100; // The integer value 100 is stored on the heap
    return ptr;
} // When createAndReturnInteger returns, 'ptr' is popped off the stack

int main() {
    int* heapInt = createAndReturnInteger();

    if (heapInt != NULL) {
        printf("Value from heap: %d\n", *heapInt); // We can access the heap data
        free(heapInt); // Free the heap memory when done
    }

    return 0;
}
```

Returning a pointer instead of simply allocating the return value of `createAndReturnInteger()` to a local variable in `main()` is useful when the value is large and, therefore, inefficient to copy. This would also save resources if passing the pointer as an argument to another function, which would avoid copying the value to another variable within the scope of those subsequent functions.

It is also useful when you need dynamic sizing, i.e., you don't know the size of the variable until runtime, or when building complex data structures. We won't go into detail on that here.

### Segfault

Failing to understand how we can interact with each of these segments can result in code that won't run as expected. A segmentation fault occurs when a disallowed operation is attempted on a particular memory segment. Examples:

- Writing to a const (stored in the read-only segment)
- Writing to the Code segment

Other kinds of illegal operations will also cause a segfault:

- Accessing an invalid memory address
- Buffer overflow (writing beyond the allocated memory of a variable)
- Stack overflow
- Accessing freed memory
- Invalid memory alignment
- Accessing unmapped memory (a memory address that doesn't exist)

**Examples for accessing an invalid memory address**

```
// Null pointer dereference
int* ptr = NULL;
*ptr = 42;  // Segfault - accessing address 0

// Uninitialised pointer
int* uninitPtr;
*uninitPtr = 100;  // Segfault - accessing random memory location

// Dangling pointer (pointer to freed memory)
int* ptr = malloc(sizeof(int));
free(ptr);
*ptr = 50;  // Segfault - accessing freed memory
```

### Stack/Heap Convergence

If the stack and/or heap grow to a point where they are contiguous, the error depends on which one attempts to write next. If it's the stack, we'll get a stack overflow; if it's the heap, it will simply fail to allocate memory. The heap allocator first checks if space is available and returns `NULL` if not, and does not cause a crash. This is why it's important to verify successful memory allocation and handle this scenario, like so:

```
#include <stdio.h>
#include <stdlib.h>

int main() {
    void* pointers[10000];
    int successfulAllocations = 0;

    for (int i = 0; i < 10000; i++) {
        pointers[i] = malloc(1024 * 1024);  // 1MB each

        if (pointers[i] == NULL) {
            printf("Successfully allocated %d MB before failure\n", successfulAllocations);
            break;
        }

        successfulAllocations++;
    }

    for (int i = 0; i < successfulAllocations; i++) {
        free(pointers[i]);
    }

    return 0;
}
```
