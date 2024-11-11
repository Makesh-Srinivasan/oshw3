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

## Testing and Evaluation

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

**Handling Out-of-Bounds Values**: The `nice.c` program and `sys_nice` system call ensure that nice values are always between 1 and 5. If a value outside this range is passed, the command fails gracefully, informing the user that the input is invalid.

### Test Results Summary

The `nice.c` utility was tested with valid and invalid inputs to ensure correct behavior. The utility worked as expected:
- **Valid Inputs**: Successfully updated the nice value, printing the previous value.
- **Invalid PIDs or Values**: Appropriately handled errors, providing clear messages to the user.

These tests confirmed the functionality and reliability of the `nice` system call in managing process priorities as intended.

