# Signals

A small message that notifies a process that a specific event has occured
in the system.  Always sent from the kernel, sometimes at the request of
another process.

## Sending a Signal

* Kernel sends a signal to a process by updatign some state in the context
  of the destination process
* kernel sends signal for follwing reasons:
    * kernel has detected a system event 
    * another process invoked the `kill` command to send a signal to the 
      destination process

## Receiving a Signal

* destination process receives a signal when it is forced to react in some way
  to the delivery of a signal
* ways to react:
    * ignore
    * terminate the process (with cleanup)
    * catch the signal and execute a user-level function called a **signal
      handler**

## Pending Signals

A signal is pending if it has been sent by a process but not yet receieved.
* There can be **at most** one pending signal of a given type
* **signals are not queued**
* processes can *block* the receipt of certain signals
* a pending signal is received at most once

Kernel maintains `pending` and `blocked` bit vectors in the context of each
process.  `pending` represents a list of the pending signals for a process,
kernel will set / clear bit k when a signal of type k is delivered / received.
`blocked` represents a bitmask over `pending` that indicates whether a 
process is blocking bit k or not.

## Signal Handlers

To do something different than the default action associated with the 
receipt of signal `signum` use:

```c
handler_t *signal(int signum, handler_t handler)
```

where:
* `signum` is the type of the signal to handle
* `handler` could be
    * `SIG_IGN` to ignore the signal
    * `SIG_DFL` to call the default handler
    * the address of a user defined handler method with the template: `void
      f(int signum)`

This is known as **installing a signal handler**.  Note that `signal` does
not signal a process, it just sets the handler for a specific signal.

## Blocking and Unblocking Signals

Kernel automatically blocks signals of the type currently being handled.  Can
also explicitly block and unblock signals using:
* `sigprocmask`
* `sigemtpyset`
* `sigfillset`
* `sigaddset`
* `sigdelset`

Temporarily Blocking Signals:
```c
sigset_t mask, prev_mask;

// Creates empty set and adds SIGINT
Sigemptyset(&mask);
Sigaddset(&mask, SIGINT);

// Tells kernel to block the signals in mask, saving previous mask to prev_mask
Sigprocmask(SIG_BLOCK, &mask, &prev_mask);

/* Code region that will not be interrupted by SIGINT */

// Restore previous mask using SIG_SETMASK
Sigprocmask(SIG_SETMASK, &prev_mask, NULL);
```

## Guidelines for Writing Safe Signal Handlers

It's easy to run into issues with buggy signal handlers, as they run 
concurrently with the main code, and share the same global data structures.  So, 
a handler can mutate shared state while other code is using that state
and can cause serious issues.  

* G0: Keep handlers as simple as possible
* G1: Call only async-signal-safe functions
* G2: Save and restore `errno` on entry / exit
* G3: Protect accesses to shared data structures by temporarily blocking
  **all** signals
* G4: Declare global variables as `volatile`
* G5: Declare global flags as `volatile sig_atomic_t`
    * a flag is a variable that is only read or written `flag = 1` NOT `flag++`
    * doesn't need to be protected like other globals

## Synchronizing Flows to Avoid Races

Consider the following example of a simple shell (with a subtle bug):

```c
void handler(int sig)
{
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t pid;

    Sigfillset(&mask_all);

    while ((pid = waitpid(-1, NULL, 0)) > 0) { /* Reap child */
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        deletejob(pid); /* Delete the child from the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    if (errno != ECHILD)
        Sio_error("waitpid error");
    errno = olderrno;
}

int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, prev_all;

    Sigfillset(&mask_all);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */
  
    while (1) {
        if ((pid = Fork()) == 0) { /* Child */
            Execve("/bin/date", argv, NULL);
        }

        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all); /* Parent */
        addjob(pid); /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    exit(0);
}
```

The issue here, is that this program assumes that the parent runs first, and
adds the job to the job queue.  However, it's entirely possible that the
child executes, finishes, triggers a `SIGCHLD` and the `handler` code gets
executed before the job can be added to the job list!  So, we have to make 
sure that the parent code to add the job to the job list is **always** executed
before the handler is called for a `SIGCHLD` signal.

Corrected Scenario (block `SIGCHLD` in the parent code before forking child):

```c
int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, prev_one, mask_sigchld;

    Sigfillset(&mask_all);
    Sigemptyset(&mask_sigchld);
    Sigaddset(&mash_sigchld, SIGCHLD);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */
  
    while (1) {
        Sigprocmask(SIG_BLOCK, &mask_sigchld, &prev_one); /* Block SIGCHLD */
        if ((pid = Fork()) == 0) { /* Child */
            Sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Restore blocked vector in the child */
            Execve("/bin/date", argv, NULL);
        }

        Sigprocmask(SIG_BLOCK, &mask_all, NULL); /* Parent */
        addjob(pid); /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_one, NULL);
    }
    exit(0);
}
```

## Waiting for Signals

we waited for fg jobs using wait
wait has to go in SIGCHLD handler, not main routine

Let's say we'd like to wait for a child process to finish, such as waiting
for a foreground job to finish before continuing execution in our parent
thread.  To do this, we use a `SIGCHLD` handler that sets a global atomic
variable `pid` of the process ID of the child.  How to wait in the parent?
Consider a naive solution:

```c
Sigprocmask(SIG_BLOCK, &mask, &prev); /* Block SIG_CHLD */
if (Fork() == 0) /* Child */
    exit(0);

pid = 0;
Sigprocmask(SIG_SETMASK, &prev, NULL); /* Unblock SIG_CHLD */
while (!pid)
    ;
```

While this does work, it's very inefficient.  The main loop is sitting here
spinning until it finds a non-zero pid.  Take a look at these other incorrect
or inefficient solutions:

This solution can run into a race condition.  Since we've unblocked `SIGCHLD`
signals, the kernel could hit us with a `SIGCHLD` at any point in this
execution.  If `pid` is not initially set, but we receive a signal before the 
call to `pause`, then `pause` will wait for a signal that will never arrive, 
since the `SIGCHLD` signal was already called
```c
while (!pid) /* Race Condition */
    pause();
```

This solution forces the system to wait a full second before checking if the 
`pid` is set.  That's forever! Even though you can choose a smaller value
to sleep for, what do you choose!
```c
while (!pid) /* Too Slow */
    sleep(1);
```
