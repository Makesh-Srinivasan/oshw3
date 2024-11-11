

# Report on Implementation of Priority-Based Scheduling in xv6

## Introduction
This report documents the implementation of a priority-based scheduling mechanism in xv6. The focus is on modifying the xv6 kernel to introduce different priority levels for processes, allowing for differentiated CPU allocation based on the concept of "nice values." The report will cover the modifications made to various parts of the xv6 codebase, including the kernel scheduling mechanism, the implementation of the `nice` system call, and the testing procedures through three custom test programs.

## Key Modifications in xv6

### 1. `nice` System Call (nice.c and sysproc.c)
The `nice` system call is implemented to allow changing the priority of processes based on a "nice value," which ranges between 1 (highest priority) and 5 (lowest priority). The following files were modified for the `nice` system call:

#### nice.c
The `nice.c` file is responsible for allowing users to set the nice value of a process from the user space. The usage format is either `nice <pid> <value>` to set the nice value for a specific process or `nice <value>` to change the nice value for the current process.

```c
#include "types.h"
#include "stat.h"
#include "user.h"

int main(int argc, char *argv[]) {
    int pid, value;

    if(argc == 2) {
        pid = getpid();
        value = atoi(argv[1]);
    } else if(argc == 3) {
        pid = atoi(argv[1]);
        value = atoi(argv[2]);
    } else {
        printf(1, "Usage: nice <pid> <value> or nice <value>\n");
        exit();
    }

    int old_value = nice(pid, value);

    if(old_value == -1) {
        printf(1, "Failed to set nice value. Check PID or value range.\n");
    } else {
        printf(1, "PID: %d, Old Nice Value: %d\n", pid, old_value);
    }
    exit();
}
```

This program handles two different use cases:
- **Setting the nice value of a specific process** (using the PID).
- **Setting the nice value of the current process**.

It outputs either the old nice value before the change or an error message if the value is invalid or the PID doesn't exist.

#### sysproc.c
In `sysproc.c`, the implementation for the `sys_nice` function was modified to retrieve the PID and new nice value, verify the validity of these parameters, and set the nice value in the corresponding process structure. The `nice` value must be between 1 and 5, with the default being 3. If the process cannot be found or the value is out of bounds, the function returns an error (-1).

```c
int sys_nice(void) {
    int pid, value;
    struct proc *p;

    if(argint(0, &pid) < 0 || argint(1, &value) < 0) {
        return -1;
    }

    if(value < 1 || value > 5) {
        return -1;
    }

    acquire(&ptable.lock);
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if(p->pid == pid) {
            int old_value = p->nice;
            p->nice = value;
            release(&ptable.lock);
            return old_value;
        }
    }
    release(&ptable.lock);
    return -1;
}
```

### 2. Modifying the Scheduler (proc.c)
The main scheduling logic in `proc.c` was modified to implement a priority-based scheduler. Processes are scheduled based on their nice value, from highest to lowest priority (i.e., nice value 1 is given precedence over nice value 5).

```c
void scheduler(void) {
    struct proc *p;
    struct cpu *c = mycpu();
    c->proc = 0;

    for(;;){
        sti();
        acquire(&ptable.lock);

        for(int priority = 1; priority <= 5; priority++){
            for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
                if(p->state != RUNNABLE || p->nice != priority) continue;

                c->proc = p;
                switchuvm(p);
                p->state = RUNNING;

                swtch(&(c->scheduler), p->context);
                switchkvm();

                c->proc = 0;
            }
        }
        release(&ptable.lock);
    }
}
```
This modification iterates over different priority levels, ensuring that runnable processes with the highest priority are selected first. This approach ensures that processes with lower nice values (higher priority) receive more CPU time compared to those with higher nice values.

## Testing and Evaluation of our codes

Three test cases (`test1.c`, `test2.c`, `test3.c`) were written to verify the implementation of the priority-based scheduling mechanism and `nice` system call. The goal of the tests was to demonstrate the impact of different nice values on process scheduling.

### Task 1: Explanation of `nice.c` and Test Cases

#### Explanation of `nice.c`
The `nice.c` utility provides an interface for setting the "nice value" (priority) of processes. There are two possible modes of operation based on the arguments provided:

1. **Set the Nice Value for a Specific PID**:
   - When two arguments are given (`nice <pid> <value>`), the utility sets the nice value for the specified process.
   - The PID and value are retrieved, and the system call `nice(pid, value)` is executed.
   - If the `nice` system call succeeds, it prints the old nice value of the process.
   - If the value is out of bounds (not between 1 and 5) or the PID is invalid, an error message is printed.

2. **Set the Nice Value for the Current Process**:
   - When only one argument is given (`nice <value>`), the utility sets the nice value for the current process using its PID.
   - Similar to the first case, the old nice value is printed if successful, otherwise an error message is shown.

The program ensures proper error handling by validating argument count and range before attempting to modify the nice value.

#### Test Cases and Output Explanation

1. **Test Case 1 - Setting a Nice Value for a Specific PID (`nice 2 4`)**:
   - **Command**: `$ nice 2 4`
   - **Expected Output**: `PID: 2, Old Nice Value: 3`
   - **Explanation**: The nice value of process with PID 2 is updated from the default value of 3 to 4. The old nice value (3) is displayed to indicate the change.

2. **Test Case 2 - Changing the Nice Value of the Current Process (`nice 2`)**:
   - **Command**: `$ nice 2`
   - **Expected Output**: `PID: 8, Old Nice Value: 3`
   - **Explanation**: This command changes the nice value of the current process to 2. The old value of 3 indicates the default nice value before adjustment. The process ID (PID) displayed corresponds to the current process.

3. **Test Case 3 - Invalid PID (`nice 999 3`)**:
   - **Command**: `$ nice 999 3`
   - **Expected Output**: `Failed to set nice value. Check PID or value range.`
   - **Explanation**: Since PID 999 does not exist, the system call fails, resulting in an error message indicating that either the PID is invalid or the value is out of range.

4. **Test Case 4 - Out-of-Bounds Nice Value (`nice 2 6`)**:
   - **Command**: `$ nice 2 6`
   - **Expected Output**: `Failed to set nice value. Check PID or value range.`
   - **Explanation**: The value provided (6) is outside the allowed range (1-5). Therefore, the system call does not proceed, and an error message is printed to indicate the issue.

**Handling Out-of-Bounds Values**: The `nice.c` program and `sys_nice` system call ensure that nice values are always between 1 and 5. If a value outside this range is passed, the command fails gracefully, informing the user that the input is invalid. Note that the four tests aboe are submitted as screenshots only and not as test1.c as it was optional. The task 2 tests are submissted as `test1.c`, `test2.c`, and `test3.c`.

### Test Results our outputs

<img width="495" alt="image" src="https://github.com/user-attachments/assets/04b66a34-70f8-48b0-bd2a-8021e9bb9631">


The `nice.c` utility was tested with valid and invalid inputs to ensure correct behavior. The utility worked as expected:
- **Valid Inputs**: Successfully updated the nice value, printing the previous value.
- **Invalid PIDs or Values**: Appropriately handled errors, providing clear messages to the user.

These tests confirmed the functionality and reliability of the `nice` system call in managing process priorities as intended.



## Task 2: Explanation and Testing of Priority Scheduler

### Test Cases for Priority Scheduler

For Task 2, three test programs (`test1.c`, `test2.c`, `test3.c`) were created to validate the implementation of the priority-based scheduling mechanism. The tests focus on ensuring that processes with different nice values receive appropriate CPU time based on their assigned priority.

### Test 1: Calculating Primes with Different Nice Values (`test1.c`)

The `test1.c` program spawns five child processes, each with a different nice value ranging from 1 to 5. Each process calculates prime numbers up to a limit and prints them.

#### Code for `test1.c`
```c
#include "types.h"
#include "stat.h"
#include "user.h"

void calculate_primes(int pid, int nice_value, int max_primes) {
    // Set the nice value for the current process
    nice(pid, nice_value);

    int count = 0;
    int n = 2;

    printf(1, "Child PID: %d, Nice Value: %d\n", pid, nice_value);

    while (count < max_primes) {
        int is_prime = 1;
        for (int i = 2; i * i <= n; i++) {
            if (n % i == 0) {
                is_prime = 0;
                break;
            }
        }
        if (is_prime) {
            printf(1, "PID: %d, Nice Value: %d, Prime: %d\n", pid, nice_value, n);
            count++;
        }
        n++;

        // Introduce a delay to simulate work
        if (count % 10 == 0) {
            sleep(1);
        }
    }
    exit();
}

int main(int argc, char *argv[]) {
    int num_processes = 5;
    int nice_values[] = {1, 2, 3, 4, 5};

    for(int i = 0; i < num_processes; i++) {
        int pid = fork();
        if(pid == 0) {
            // Child process
            int nice_value = nice_values[i];
            int max_primes = 50;
            calculate_primes(getpid(), nice_value, max_primes);
        }
    }

    // Parent waits for all child processes to finish
    for(int i = 0; i < num_processes; i++) {
        wait();
    }
    exit();
}
```

#### Expected Output for `test1.c`
The processes with lower nice values should receive more CPU time and hence calculate more prime numbers in a given period. The expected output will show:
- **Child processes with nice values 1 and 2** will print more primes compared to those with values 4 and 5.
- This validates the priority-based scheduling since processes with higher priorities (lower nice values) are given preference.

#### Output of test1.c:

<img width="662" alt="image" src="https://github.com/user-attachments/assets/4a71bab6-ad40-4239-a27c-575d13204289">

Here, the output is cropped slightly and the outputs are unsynchronised because of the forks. But through manual verification, it is clear that the lower nice values receive more CPU time and hence calculate more prime numbers in a given period.

An example of a synchronised expected output will be like this with more CPU time for lower nice values.

![image](https://github.com/user-attachments/assets/6588e9e9-7e01-467c-865f-9b607b713cd7)



### Test 2: Identical Nice Values for All Processes (`test2.c`)

The `test2.c` program creates three child processes, each assigned the same nice value of 3. The processes then calculate a fixed number of prime numbers and print their results.

#### Code for `test2.c`
```c
// test2.c
#include "types.h"
#include "stat.h"
#include "user.h"

void calculate_primes(int pid, int nice_value, int max_primes) {
    int count = 0;
    int n = 2;

    printf(1, "Child PID: %d, Nice Value: %d\n", pid, nice_value);

    while (count < max_primes) {
        int is_prime = 1;
        for (int i = 2; i * i <= n; i++) {
            if (n % i == 0) {
                is_prime = 0;
                break;
            }
        }
        if (is_prime) {
            printf(1, "PID: %d, Prime: %d\n", pid, n);
            count++;
        }
        n++;
    }
    exit();
}

int main(int argc, char *argv[]) {
    int num_processes = 3;
    int nice_value = 3; // All processes have the same nice value
    int max_primes = 20;

    for (int i = 0; i < num_processes; i++) {
        int pid = fork();
        if (pid == 0) {
            // Child process
            int child_pid = getpid();
            nice(child_pid, nice_value);
            calculate_primes(child_pid, nice_value, max_primes);
        }
    }

    // Parent waits for all child processes to finish
    for (int i = 0; i < num_processes; i++) {
        wait();
    }
    exit();
}
```

#### Expected Output for `test2.c`
Since all processes have the same nice value, the CPU time is equally divided among them. The output should indicate that all three processes completed roughly the same amount of work (i.e., calculating the same number of primes).

<img width="288" alt="image" src="https://github.com/user-attachments/assets/aac07a9d-9bff-46cf-8a20-a198039a7463">

Three Child Processes: The output begins with the lines indicating that three child processes were spawned, each with a nice value of 3.
```
Child PID: 18, Nice Value: 3
Child PID: 19, Nice Value: 3
Child PID: 20, Nice Value: 3
```
Garbled Output: The prime number calculations are printed, but they appear to be mixed and unsynchronized. There are multiple instances where the characters are jumbled together, likely because all three processes were printing to the console at the same time. 

Since all processes have the same nice value, they are given equal opportunity to run. This means that the scheduler keeps switching between the processes, and they each get time slices for execution.As a result, the printf statements from different processes are being executed concurrently, leading to overlapping text and jumbled characters in the output. This is because printf operations are not atomic—they can be interrupted in the middle, causing one process's output to mix with that of another. 


### Test 3: Different Priorities for Simple Iterative Work (`test3.c`)

The `test3.c` program spawns three child processes, each with a different nice value (1, 3, 5). Each process performs a simple iterative task of printing a message multiple times, demonstrating how scheduling is influenced by nice values.

#### Code for `test3.c`
```c
#include "types.h"
#include "stat.h"
#include "user.h"

void simple_work(int pid, int nice_value, int limit) {
    for (int i = 0; i < limit; i++) {
        printf(1, "PID: %d, Nice Value: %d, Iteration: %d\n", pid, nice_value, i + 1);
    }
    printf(1, "PID: %d, Nice Value: %d, Completed\n", pid, nice_value);
    exit();
}

int main(int argc, char *argv[]) {
    int pid1, pid2, pid3;

    // Fork first child process with high priority
    pid1 = fork();
    if (pid1 == 0) {
        int child_pid = getpid();
        nice(child_pid, 1); // Highest priority
        simple_work(child_pid, 1, 5); // Perform simple work for 5 iterations
    }

    // Fork second child process with medium priority
    pid2 = fork();
    if (pid2 == 0) {
        int child_pid = getpid();
        nice(child_pid, 3); // Medium priority
        simple_work(child_pid, 3, 5); // Perform simple work for 5 iterations
    }

    // Fork third child process with low priority
    pid3 = fork();
    if (pid3 == 0) {
        int child_pid = getpid();
        nice(child_pid, 5); // Lowest priority
        simple_work(child_pid, 5, 5); // Perform simple work for 5 iterations
    }

    // Parent waits for all child processes to complete
    wait();
    wait();
    wait();

    exit();
}
```

#### Expected Output for `test3.c`
The process with the highest priority (nice value 1) should complete its iterations first, followed by the medium priority process (nice value 3), and lastly the low priority process (nice value 5). This output order demonstrates the effectiveness of the priority scheduler.

<img width="683" alt="image" src="https://github.com/user-attachments/assets/203bdc2a-7729-4069-9aad-3535ee00a835">

From this output, the expected priority order was followed: PID 22 (highest priority) completed first, followed by PID 23, and then PID 24 (lowest priority). The final output reflects that higher-priority processes received CPU time before the lower-priority ones. The unsynchronized and overlapping text is an expected side effect when multiple processes print to the console without any locking mechanism to ensure orderly output.

### Test Results of our tests
The results from the three test cases validate that the priority-based scheduler is working as intended:
- **Higher priority processes** consistently received more CPU time and completed their tasks earlier.
- **Lower priority processes** had to wait for higher priority ones to finish, as indicated by delayed or slower progress in their outputs.

These observations confirm that the scheduler effectively differentiates CPU allocation based on nice values, ensuring that processes with higher priority are favored for execution.


