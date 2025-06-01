# EE_Hardware_Interview
These are some questions and answers that you should know before going into an interview.
## Questions summary list
1. **volatile:** Why do we mark a variable volatile, and when does it still fail us?
2. **Interrupt to main loop:** How do you pass one byte from an ISR to the main code without a race? 
3. **Power-on path:** Walk me from the reset pin to the first line in main().
4. **Stack math:** What is stack roll and unroll and when does stack gets initialized?
5. **Hard fault:** The MCU halts. Tell me the first three registers you check and why?

## Question 1 - volatile: Why do we mark a variable volatile, and when does it still fail us? 
The `volatile` keyword in C/C++ tells the compiler that a variable's value can change unexpectedly, preventing certain optimizations. Here's when and why to use it:

### Why Mark Variables Volatile
1. **Prevents compiler optimizations** that assume the variable won't change
2. **Forces memory reads/writes** on every access
3. **Prevents caching** the variable in CPU registers

### Common Use Cases
- **Memory-mapped I/O registers** - Hardware can change values
- **Variables modified by interrupt handlers**
- **Multi-threaded shared data** (though this is problematic)
- **Variables modified by signal handlers**

### When Volatile Fails Us

#### 1. **Not Atomic**
```c
volatile int counter = 0;
// This is NOT thread-safe!
counter++;  // Read-modify-write is not atomic
```
#### 2. **No Memory Ordering Guarantees**
```c
volatile bool ready = false;
volatile int data = 0;

// Thread 1
data = 42;
ready = true;  // No guarantee this happens after data assignment

// Thread 2
if (ready) {
    use(data);  // Might see old value of data
}
```
#### 3. **Compiler Reordering**
The compiler can still reorder volatile accesses with non-volatile operations.
#### 4. **CPU Reordering**
Modern CPUs can reorder memory operations despite volatile.
### Better Alternatives
- **Atomic operations** (`std::atomic` in C++, `_Atomic` in C11)
- **Memory barriers/fences**
- **Mutex/locks** for thread synchronization
- **Memory-mapped I/O libraries** for hardware access

**Bottom line:** `volatile` is mainly for hardware registers and signal handlers. For multi-threading, use proper synchronization primitives instead.


## Question 2 - Interrupt to main loop: How do you pass one byte from an ISR to the main code without a race? 
To pass one byte from an ISR to main code without a race condition, you need to ensure atomic access. Here are the most common approaches:

### **1. Volatile Variable (Simple but Limited)**
```c
volatile uint8_t isr_data;

// In ISR
void interrupt_handler(void) {
    isr_data = read_peripheral();
}

// In main
int main(void) {
    uint8_t local_copy = isr_data;  // Single byte read is atomic on most platforms
    // Process local_copy...
}
```

### **2. Flag-Based Apprach (Recommended)**
```c
volatile uint8_t isr_data;
volatile bool data_ready = false;

// In ISR
void interrupt_handler(void) {
    isr_data = read_peripheral();
    data_ready = true;  // Set flag last
}

// In main
int main(void) {
    if (data_ready) {
        uint8_t local_copy = isr_data;
        data_ready = false;  // Clear flag after reading
        // Process local_copy...
    }
}
```

### **3. Disable Interrupts During Access**
```c
volatile uint8_t isr_data;

// In main
int main(void) {
    uint8_t local_copy;
    
    __disable_irq();  // Or CLI() on AVR
    local_copy = isr_data;
    __enable_irq();   // Or SEI() on AVR
    
    // Process local_copy...
}
```

### **Key Points:**
- **Single byte reads/writes are typically atomic** on 8-bit and larger processors
- Always use `volatile` for variables shared between ISR and main code
- The flag-based approach is safest and most portable
- For multi-byte data, you'll need additional synchronization (circular buffers, disable interrupts, etc.)

The flag-based approach (#2) is generally recommended as it's explicit about data freshness and works reliably across different architectures.


## Question 3 - Power-on path: Walk me from the reset pin to the first line in main(), answer in general/as a hypothetical. Link the reset pin to the first line of any main in any embedded code.
### Boot Sequence: Reset Pin to main()
Here's the general flow from reset pin activation to the first line of main() in embedded systems:

#### **1. Reset Pin Assertion**
- Hardware reset pin is pulled low (or high, depending on architecture)
- CPU internal state is cleared (registers, flags, caches)
- Program counter (PC) is loaded with reset vector address
#### **2. Reset Vector Execution**
- CPU fetches instruction from reset vector (typically address 0x00000000 or 0xFFFFFFFC)
- This usually contains a jump instruction to startup code
```as
// Reset vector table (ARM Cortex-M example)
.section .vector_table
.word _stack_top          // Initial stack pointer
.word Reset_Handler       // Reset vector
.word NMI_Handler         // NMI handler
// ...more vectors
```

#### ** 3. Startup/Boot Code**
The reset handler performs critical initialization:
```c
void Reset_Handler(void) {
    // Set up stack pointer
    __set_MSP((uint32_t)&_stack_top);
    
    // Initialize system clock
    SystemInit();
    
    // Copy initialized data from flash to RAM
    // ...existing code...
    
    // Zero out uninitialized data (BSS section)
    // ...existing code...
    
    // Call main()
    main();
}
```
#### **4. System Initialization**
- **Clock setup** - Configure system clocks, PLLs
- **Memory setup** - Initialize RAM, configure memory controllers
- **Data initialization** - Copy .data section from flash to RAM
- **BSS clearing** - Zero out uninitialized variables
#### **5. C Runtime Initialization**
- Global constructors called (C++)
- Runtime library initialization
- Standard library setup (if used)
#### **6. Jump to main()**
Finally, control transfers to your application:
```c
int main(void) {
    // <-- First line executes here
    // Your application code starts
    while(1) {
        // Application loop
    }
}
```
### **Key Points:**
- **Hardware dependency** - Exact sequence varies by processor (ARM, AVR, PIC, etc.)
- **Toolchain role** - Linker script defines memory layout and startup sequence
- **Boot time** - Entire process typically takes microseconds to milliseconds
- **Deterministic** - Same sequence every reset for reliable embedded operation
This sequence ensures the processor transitions from an unknown hardware state to a predictable software environment where your main() function can execute reliably.

## Question 4 - Stack Roll and Unroll: What is stack roll and unroll and when does stack gets initialized?

**Stack Roll (Push)** and **Stack Unroll (Pop)** refer to the fundamental operations of adding and removing data from a stack data structure:

#### Stack Roll (Push)
- **Operation**: Adding an element to the top of the stack
- **Effect**: Increases stack size by one
- **Time Complexity**: O(1)

#### Stack Unroll (Pop)
- **Operation**: Removing the top element from the stack
- **Effect**: Decreases stack size by one, returns the removed element
- **Time Complexity**: O(1)

### Stack Initialization

Stacks get initialized at different times depending on the context:

#### 1. **Program Stack (Call Stack)**
- Initialized when a program starts execution
- Each thread gets its own stack
- Size is typically set by the OS (usually 1-8 MB on Windows)

#### 2. **Data Structure Stack**
- Initialized when you create a stack object in your code
- Can be dynamic (linked list) or static (array-based)

#### Example Implementation:

```python
class Stack:
    def __init__(self):
        """Stack initialization"""
        self.items = []
    
    def push(self, item):
        """Stack roll operation"""
        self.items.append(item)
    
    def pop(self):
        """Stack unroll operation"""
        if not self.is_empty():
            return self.items.pop()
        raise IndexError("Pop from empty stack")
    
    def is_empty(self):
        return len(self.items) == 0

# Stack gets initialized here
my_stack = Stack()
my_stack.push(10)  # Roll
my_stack.push(20)  # Roll
value = my_stack.pop()  # Unroll (returns 20)
```

### 3. **Function Call Stack**
- New stack frame created each time a function is called
- Destroyed when function returns
- Contains local variables, parameters, and return addresses

The stack follows **LIFO (Last In, First Out)** principle - the last element pushed is the first one popped.

## Question 5 - MCU Halt Debugging: First Three Registers to Check

When an MCU halts unexpectedly, here are the first three registers I'd check:

### 1. **Program Counter (PC)**
- **Why**: Shows exactly where the processor stopped executing
- Helps identify if the halt occurred at a specific instruction, in an interrupt handler, or at an unexpected memory location
- Can reveal if the processor jumped to invalid code or got stuck in a loop

### 2. **Stack Pointer (SP)**
- **Why**: Stack corruption is a common cause of MCU halts
- Check if SP points to valid RAM or if it's corrupted (pointing to invalid memory)
- Stack overflow/underflow can cause unpredictable behavior and system crashes

### 3. **Status/Control Register (often called PSR, SR, or similar)**
- **Why**: Contains critical processor state information
- Shows interrupt enable/disable flags
- Indicates processor mode (user/supervisor)
- Contains condition flags that might reveal the last operation before halt
- Can show if the processor is in an exception or fault state

These three registers provide immediate insight into **where** the failure occurred (PC), **memory integrity** (SP), and **processor state** (Status Register), giving you the foundation to diagnose the root cause of the halt.

# Source of questions
1. https://www.linkedin.com/posts/balemarthyvamsi_if-you-cant-answer-these-5-questions-i-activity-7334579430817714176-VOQ-
