# User Level Thread Switch

This article will introduce how CPU handle function call and how to do user level thread switch.

## Function Call
Let's image a simple computer, all instructions are stored in memory, and CPU will execute the instruction on by one in sequence.

As we want to pick an instruction to execute, we need to know the instruction's address in memory.

So we have to find a place to hold such info for CPU, CPU uses registers for this purpose.

The registers are much more faster than memory, accessing a register may need one CPU cycle and accessing a memory location may need hundred CPU cycles.

But CPU just has limited registers, may 32 or 64 or more, but can't get as much as we want.

The register for storing next instruction's address is `PC`.

After we executed one instruction, we move the `PC` 4 byte forward (suppose 32 bits for each instruction).

Then CPU will execute the instruction which is point by `PC`.

We can describe this in code

``` c
instructions = [i0, i1, ....in]

pc = 0
while true
    CPU_exec(instructions[pc])
    pc += 4
```
<img width="570" alt="Screen Shot 2021-07-28 at 8 45 34 AM" src="https://user-images.githubusercontent.com/3775525/127245579-1c7def90-a730-49d8-8ec5-1367f07e92af.png">

Now let's to see how loop works.

For loop we need exec some instructions again and agin, such as instruction-0, instruction-1, .... instruction-n, then instruction-0, instruction-1, .... instruction-n...

As we know `PC` is pointed to the next instruction to execute, so after we executed instruction-n, we just need to set `PC` to the instruction-0's address, then cpu will loop again.

<img width="223" alt="Screen Shot 2021-07-28 at 8 52 01 AM" src="https://user-images.githubusercontent.com/3775525/127246041-86e12d78-f2a6-4f7c-8a3d-a4129600f8f5.png">

It's time for function call.

Suppose there is some code as below:

``` c
1 foo :=
2  instruction0
3  call bar
4  instruction1
5  instruction2
6
7 bar :=
8  instruction0
9  ret
```

For function `foo`, we need execute some instruction then call `bar` function and execute remain instructions.

For function `bar`, we need execute some instruction and return.

<img width="412" alt="Screen Shot 2021-07-28 at 9 10 21 AM" src="https://user-images.githubusercontent.com/3775525/127247529-dae03166-0b96-433e-b666-5316a0bb6573.png">

So we need jump to `bar` and jump back.

When we jump to `bar`, we just need to update `PC` to `bar`'s location. Then CPU will execute `bar`'s instructions.

After we finished `bar`'s execution code we need jump back to line 4's address.

For doing this we need one place to hold the line 4's address (for jumping back).

We can store it in memory but memory is slow, so we store it in register. This special purpose register is called `ra` usually.

so the call and ret can be defined as below:

``` c
call label :=
  ra <- pc + 4 // assign next instruction for ret, which is line 4's address in our example
  pc <- label  // jump to the callee, for foo is bar's address

ret :=
  pc <- ra // jump back
```

It works for on level function call.

But what if we have 3 functions? Likes function `f` calls `foo` and `foo` calls `bar`.

``` c
1  0000  f :=                    exec order       ra's value      description
2  0004    instruction-x            0                 0
3  0008    call foo                 1                 000a
4  000a    instruction-x
5  000e    ret
6  0010  foo :=
7  0014    instruction-x            2                 000a
8  0016    call bar                 3                 0001e
9  001e    instruction-x            6                 001e
10 0010    ret                      7                 001e        jump to 001e, line 9 again...
11 0014  bar :=
12 0018    instruction-x            4                 001e
13 001a    ret                      5                 001e         jump to 001e, line 9
```

Let's walk above code step by step.

For line 3, the ra's value is 000a(at line 3), then we jump to foo function(at line 6).

Then we execute `call bar` at line 8 and the ra's value will be updated to 001e(at line 9).

After `bar` has been executed, `ra`'s value is 001e(at line 9), so we jump to line 9.

But when we execute line 10, current `ra`'s value is 001e(at line 9), we jump back to line 9 again.

It's an infinite loop.

That happens because at line 8 we overwrite the original `pa`'s value.

We need many places to store the ongoing functions' return addresses but we only have limited register.

It is impossible to store all those return address into registers, so we store them in memory.

For each function call we need some small memory to save `ra` and after the function has been executed we will need get the data back and free the memory.

For function `f` we need allocate memory and store 000e(line 3) to it and for `foo` we need allocate memory and store 0001e(at line 9),

after `bar` has been executed then we need get last value(0001e at line 9) back and deallocate memory 

then do same thing for value 000e(line 3).

The operations are push, push and pop, pop, so we can use stack.

For stack we need allocates some memory(eg: 4kb) and one pointer which points to the top of the stack.

When we do push, we move the pointer forward and store `ra` to the last location. 

After a function has been called, we pop the value and move the pointer backward.

As this happens so often, hardware designer provides one register for us, it is called `sp` usually.

Register `ra`'s value only has meaning in current function, when we executed an inner function such as `bar`, the `ra` should have different value. 

It is a temporary register, such registers should be saved by caller.

And there is another kind of register which are preserved across function calls, 

so if callee needs to use such register, he needs to save them and restore them back before return to caller.

This kind of registers are callee saved registers.

Above is how we handle function call.

## Switch Threads

NOTICE: Those are based on MIT 6.S081 2020 Multithreading lab.

We need build a function for switching two thread. 

And the function will be called likes `switch(thread1, thread2)`, thread1 is a variable which stores thread 1's state.

When call `switch` function we will switch from `thread1` to `thread2`.

![Screen Shot 2021-07-28 at 9 54 12 PM](https://user-images.githubusercontent.com/3775525/127334554-fa2caa8f-dc62-41f0-a6ae-17078391a623.png)

When we call `switch`, CPU will begin execute `switch` instructions.

And after all `switch`'s instructions have been executed, switch will not jump back to is callee place in thread 1, but will jump to other place in thread 2.

It works as same as function call except it doesn't return back.

For normal function all, compiler will help us to save register and handle return address.

For switch, we need handle those stuff by myself.

First thing is store current thread's state. Suppose we have structures as below:

```c
struct context {
  uint64 ra;
  uint64 register1;
  uint64 register2;
  .........
}

struct thread {
  struct context context;
};
```

and we call `switch` by `switch(&threa_1->context, &thread_2->context)`.

We need use assembly code to save thread_1's return address and other registers,

```c
switch:
    // a0 is thread 1's context address
   sd ra,        0(a0)     // save thread_1->context.ra into ra register
   sd register1, 8(a0)    // save thread_2->context.register1 into register1
   sd register2, 16(a0)  // save thread_2->context.register2 into register2
   ...
```

Then we need to load thread_2's context into those registers
```
   // a1 is thread 2's context address
   ld ra,         0(a1)  // load thread_2->context.ra into ra register
   ld register1,  8(a1)  // load thread_2->context.register1 into register1
   ld register2, 16(a1) // load thread_2->context.register2 into register2
```

Above code will allow us switch thread1 to thread2, but it doesn't work correlty.

Because we can have many threads but only have one stack.

We need different stacks for different user level threads and when we switch thread we need switch stack too.

So let add one field for storing the stack, the thread structure becomes:

```c
struct thread {
  char       stack[MAX_STACK_SIZE]; // toy code, do not handle stack overflow
  struct     context context;
};
```

and as we have different stack, we need add sp to `context`

```c
struct context {
  uint64 ra;
  uint64 sp;
  uint64 register1;
  uint64 register2;
  .........
}
```

and `switch` becomes:

```c
switch:
   // a0 is thread 1's context address
   sd ra,        0(a0)   // save thread_1->context.ra into ra register
   sd sp,        8(a0)    // save thread_1->context.sp into sp register
   sd register1, 16(a0)    // save thread_1->context.register1 into register1

   ...

   // a1 is thread 2's context address
   ld ra,        0(a1)   // load thread_2->context.ra into ra register
   ld sp,        8(a1)    // load thread_2->context.ra into sp register
   ld register1, 16(a1)  // load thread_2->context.register1 into register1

   ...
```

When we create a user level thread, we need set the stack's address for the thread's `context.sp` field and set `context.ra` to the function we want to execute.

So when we switch to the thread, CPU will jump to the `thread->context.ra` and use `thread->context.sp` as its stack.

The memory layout as below:

![Screen Shot 2021-07-29 at 9 20 34 AM](https://user-images.githubusercontent.com/3775525/127416803-5ca48c79-d27e-43e2-9027-5d77f9b157b8.png)

From this section, we know if we want to have user level thread, 

we need to allocate memory for each user level thread's stack and handle `ra` and other callee registers in `switch` function.

That's how we handle user level thread switching.

## References

- [What are callee and caller saved registers?](https://stackoverflow.com/questions/9268586/what-are-callee-and-caller-saved-registers/16265609#16265609)

- [MIT 6.S081 2020 Multithreading lab](https://pdos.csail.mit.edu/6.S081/2020/labs/thread.html)
