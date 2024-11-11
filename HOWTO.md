# Running the xv6 Priority Scheduler and Test Cases

This guide explains how to run the modified xv6 system with the implemented priority-based scheduler, the `nice` system call, and the related test programs. Follow the steps below to compile xv6, run the virtual machine, and execute the provided test cases.

## Step 1: Compiling xv6
To compile xv6 with the modifications for the priority-based scheduler and the `nice` system call, execute the following commands:

```sh
$ make
$ make clean
$ make
$ make qemu-nox
```

- **`make`**: Compiles the xv6 operating system.
- **`make clean`**: Cleans up previous build files to avoid compilation issues.
- **`make`**: Compiles a fresh build.
- **`make qemu-nox`**: Boots xv6 using the QEMU emulator in no-X mode (no graphical interface, console output only).

After running these commands, you will enter the xv6 environment.

## Step 2: Running the `nice` Utility Test Cases

The `nice` system call allows changing the priority of processes based on a "nice value". The following four test cases demonstrate the usage of the `nice` command in xv6.

### Nice Command Test Cases

1. **Set the Nice Value for a Specific PID**
   ```sh
   $ nice 2 4
   ```
   This command sets the nice value of process with PID 2 to 4. If successful, it will print the old nice value.

2. **Set the Nice Value for the Current Process**
   ```sh
   $ nice 2
   ```
   This command sets the nice value for the current process to 2.

3. **Attempt to Set an Out-of-Bounds Nice Value**
   ```sh
   $ nice 2 6
   ```
   This command tries to set the nice value of process with PID 2 to 6, which is out of the valid range (1-5). An error message should be returned.

4. **Attempt to Set a Nice Value for a Non-Existent PID**
   ```sh
   $ nice 999 3
   ```
   This command attempts to set the nice value of a non-existent process (PID 999). An error message should indicate that the PID is invalid.

## Step 3: Running Priority Scheduler Test Programs

Three test programs (`test1`, `test2`, and `test3`) are provided to demonstrate the impact of the priority-based scheduler on different processes. These programs illustrate how processes with different nice values receive CPU time.

### Test Programs for Task 2

1. **Test 1: Calculating Primes with Different Nice Values**
   ```sh
   $ test1 2 3
   ```
   This test spawns five child processes, each with a different nice value (from 1 to 5). Each child calculates prime numbers, demonstrating how higher priority processes (lower nice values) receive more CPU time.

2. **Test 2: Identical Nice Values for All Processes**
   ```sh
   $ test2
   ```
   This test spawns three child processes, all with the same nice value of 3. The output will show that all processes receive an equal share of CPU time.

3. **Test 3: Different Priorities for Simple Iterative Work**
   ```sh
   $ test3
   ```
   This test spawns three child processes with nice values of 1, 3, and 5, respectively. Each process performs a simple task of printing iterations. Processes with higher priority (lower nice values) should complete their iterations first.

## Notes
- All commands are to be executed **inside the xv6 environment**.

A work by Makesh Srinivasan and Vaishnavi
