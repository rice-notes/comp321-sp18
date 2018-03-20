# Basics of ECF and Child Processes

### Reaping Zombie Processes:

After forking and launching a child, the parent needs to keep track of the
children.  After the child terminates, the parent must reap the finished
process, otherwise it uses system resources.  If the parent exits, the child's
parent is then updated to the init process by the OS, which will then remove
it.

How to reap? A simple way is for the parent to simply `wait` on the child:
```c

int main()
{
        int pid;
        if ((pid = fork()) == 0) {
                /* Child */
                exit(0);
        } else {
                /* Parent waits on child */
                wait();
        }
}

```

### Loading and Running Programs

After forking, the child is simply running the exact code as the parent.  To fork and
run new programs, we use `execve`:

```c
int execve(char *filename, char *argv[], char *envp);
```

* `filename`: path to object or script file
* `argv` argument list (`argv[0] == filename`)
* `envp` environment variable list

Notes on `execve`:
* complete overrites code, data, and stack
* retains PID

Why have two separate commands for forking and running a program?
* allows us to run code in child process before making call to `execve`
* specifically, it allows us to block certain signals from the OS



