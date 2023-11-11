## Some oddities of sigaction and SA_ONSTACK

If a C++ program accesses unmapped memory, it will raise the signal SIGSEGV, which will (by default) terminate the process and dump core. However, the application can install a signal handler with POSIX's [sigaction](https://man7.org/linux/man-pages/man2/sigaction.2.html) which will be invoked instead. This signal handler is [very](https://man7.org/linux/man-pages/man7/signal-safety.7.html) [restricted](https://en.cppreference.com/w/cpp/utility/program/signal) in what it can do, since it can run by interrupting essentially any code in the program. Even so, signal handlers are very useful for collecting basic diagnostic information before terminating the process. For these examples, I ignore most of these restrictions since we know that the signal is not being raised from code holding locks, i.e. inside `malloc()`.

Here is a basic example of installing a signal handler for SIGSEGV and then triggering a SIGSEGV. As expected, we dereference a null pointer, and the signal handler runs. The signal handler must terminate the process. Returning would be undefined behavior, though `longjump` can be used in some circumstances.
```
#include <iostream>
#include <signal.h>
#include <cassert>
#include <stdlib.h>

void signalHandler(int sig, siginfo_t *info, void *ucontext) {
    std::cout << "In signalHandler" << std::endl;
    ::_Exit(1);
}

int main() {
    struct sigaction sigact;
    sigact.sa_sigaction = signalHandler;
    sigact.sa_flags = SA_SIGINFO;
    int ret = sigaction(SIGSEGV, &sigact, nullptr);
    assert(ret == 0);
    std::cout << "Causing a critical signal" << std::endl;
    *(char*)nullptr = 0;
    return 0;
}
```
```
Causing a critical signal
In signalHandler
<Program terimates with code 1>
```
[godbolt](https://godbolt.org/z/KadYdeeqo)

So this works for when dereferencing a null pointer. What about a more complicated sort of SIGSEGV - stack overflow? By default, signal handlers run on the same stack as the thread which caused the signal (at least for thread specific signals like SIGSEGV and SIGFPE), so the signal handler will immediately stack overflow itself, without being able to produce any kind of diagnostic.
```
#include <iostream>
#include <signal.h>
#include <cassert>
#include <stdlib.h>

void signalHandler(int sig, siginfo_t *info, void *ucontext) {
    std::cout << "In signalHandler" << std::endl;
    ::_Exit(1);
}

void overflowTheStack() {
    overflowTheStack();
}

int main() {
    struct sigaction sigact;
    sigact.sa_sigaction = signalHandler;
    sigact.sa_flags = SA_SIGINFO;
    const auto ret = sigaction(SIGSEGV, &sigact, nullptr);
    assert(ret == 0);
    std::cout << "Causing a stack overflow" << std::endl;
    overflowTheStack();
    return 0;
}
```
```
Causing a stack overflow
<Program terimates with SIGSEGV>
```
[godbolt](https://godbolt.org/z/KcaPzPnfx)

POSIX provides a tool to deal with this. For each thread, you can install an alternate signal handler stack with [`sigaltstack`](https://man7.org/linux/man-pages/man2/sigaltstack.2.html), which is special space reserved for just the signal handler. You can then register the signal handler with the `SA_ONSTACK` flag to `sigaction`, which causes the signal handler to run in this dedicated stack memory, even if the original stack was exhausted by a stack overflow.
```
#include <iostream>
#include <signal.h>
#include <cassert>
#include <stdlib.h>

void signalHandler(int sig, siginfo_t *info, void *ucontext) {
    std::cout << "In signalHandler" << std::endl;
    ::_Exit(1);
}

void overflowTheStack() {
    overflowTheStack();
}

int main() {
    int ret;

    constexpr size_t stackSize = 64 * 1024;
    void* stackMemory = malloc(stackSize);
    assert(stackMemory != nullptr);
    stack_t newStack{.ss_sp = stackMemory, .ss_flags = 0, .ss_size = stackSize};
    ret = sigaltstack(&newStack, nullptr);
    assert(ret == 0);

    struct sigaction sigact;
    sigact.sa_sigaction = signalHandler;
    sigact.sa_flags = SA_SIGINFO | SA_ONSTACK;
    ret = sigaction(SIGSEGV, &sigact, nullptr);
    assert(ret == 0);
    std::cout << "Causing a stack overflow" << std::endl;
    overflowTheStack();
    return 0;
}
```
```
Causing a stack overflow
In signalHandler
<Program terminates with code 1>
```
[godbolt](https://godbolt.org/z/6zcxsEqzs)

Signal handlers are code, and code has bugs. What if the signal handler itself causes a SIGSEGV? The default behavior with `sigaction` is to block the signal which caused the signal handler for the duration of the signal handler. In the case of `SIGSEGV`, causing the signal while it is blocked immediately terminates the process. This is sensible - the alternative scenario, where the signal isn't blocked, would cause an infinite loop where the signal handler invokes itself forever.
```
#include <iostream>
#include <signal.h>
#include <cassert>
#include <stdlib.h>

void overflowTheStack() {
    overflowTheStack();
}

void signalHandler(int sig, siginfo_t *info, void *ucontext) {
    int stackVariable;
    std::cout << "In signalHandler at " << &stackVariable << std::endl;
    *(char*)nullptr = 0;
    _Exit(1);
}

int main() {
    int ret;

    constexpr size_t stackSize = 64 * 1024;
    void* stackMemory = malloc(stackSize);
    assert(stackMemory != nullptr);
    stack_t newStack{.ss_sp = stackMemory, .ss_flags = 0, .ss_size = stackSize};
    ret = sigaltstack(&newStack, nullptr);
    assert(ret == 0);

    struct sigaction sigact;
    sigact.sa_sigaction = signalHandler;
    sigact.sa_flags = SA_SIGINFO | SA_ONSTACK;
    ret = sigaction(SIGSEGV, &sigact, nullptr);
    assert(ret == 0);
    std::cout << "Causing a stack overflow" << std::endl;
    overflowTheStack();
    return 0;
}
```
```
Causing a stack overflow
In signalHandler at 0x10b5d6c
<Program terminates with SIGSEGV>
```
[godbolt](https://godbolt.org/z/71PxWd3PM)


If for some reason we *want* to invoke the signal handler again if it segfaults, we can. The `SA_NODEFER` flag to `sigaction` overrides the default behavior, and leaves the signal unblocked during the signal handler. Alternatively, unblock the signal by calling [`sigprocmask`](https://man7.org/linux/man-pages/man2/sigprocmask.2.html) in the signal handler. If we segfault in the signal handler, it gets invoked again. But that leaves an open question - what memory does the second invocation of the signal handler run on? The original stack? The alternate stack? It turns out, it runs on the alternate stack, but *after* (smaller address) the memory already used for the first invocation of the signal handler. This means that with each invocation of the signal handler, we have less and less stack space left, until eventually we overflow the stack in the signal handler. This causes another SIGSEGV, but for some reason, this one terminates the process rather than invoking the signal handler again forever.
```
#include <iostream>
#include <signal.h>
#include <cassert>
#include <stdlib.h>
#include <asm/ucontext.h>

void overflowTheStack() {
    overflowTheStack();
}

void signalHandler(int sig, siginfo_t *info, void *uc) {
    ucontext_t* ucontext = reinterpret_cast<ucontext_t*>(uc);

    // The lowest address in the stack
    void* stackLocation = ucontext->uc_stack.ss_sp;

    // A variable within the current stack frame
    char stackVariable;

    // On Linux, the stack grows down towards smaller addresses
    size_t stackRemaining = &stackVariable - reinterpret_cast<char*>(stackLocation);

    std::cout << "On signal stack at " << stackLocation << " with " << stackRemaining << " bytes of stack memory remaining" << std::endl;
    
    // Cause a critical signal in the signal handler
    *(char*)nullptr = 0;
}

int main() {
    int ret;

    constexpr size_t stackSize = 16 * 1024;
    void* stackMemory = malloc(stackSize);
    assert(stackMemory != nullptr);
    stack_t newStack{.ss_sp = stackMemory, .ss_flags = 0, .ss_size = stackSize};
    ret = sigaltstack(&newStack, nullptr);
    assert(ret == 0);

    struct sigaction sigact;
    sigact.sa_sigaction = signalHandler;
    sigact.sa_flags = SA_SIGINFO | SA_ONSTACK | SA_NODEFER;
    ret = sigaction(SIGSEGV, &sigact, nullptr);
    assert(ret == 0);
    std::cout << "Causing a stack overflow" << std::endl;
    overflowTheStack();
    return 0;
}
```
```
Causing a stack overflow
On signal stack at 0x42d2b0 with 15015 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 13479 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 11943 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 10407 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 8871 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 7335 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 5799 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 4263 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 2727 bytes of stack memory remaining
On signal stack at 0x42d2b0 with 1191 bytes of stack memory remaining
<Program terminates with SIGSEGV>
```
[godbolt](https://godbolt.org/z/6KvzTq7Pa)


Now what happens if we overflow the stack in the signal handler? Based on the example above, you might think it would terminate the process, since it seems that when the signal handler runs out of space that SIGSEGV doesn't cause the signal handler to run. However, in this case, it invokes the signal handler again at the same position on the alternate stack as the first time - not after the first invocation. That means the amount of stack memory doesn't decrease, and it really is an infinite loop, at least on some platforms.
```
#include <iostream>
#include <signal.h>
#include <cassert>
#include <stdlib.h>
#include <asm/ucontext.h>

void overflowTheStack() {
    overflowTheStack();
}

void signalHandler(int sig, siginfo_t *info, void *uc) {
    ucontext_t* ucontext = reinterpret_cast<ucontext_t*>(uc);

    // The lowest address in the stack
    void* stackLocation = ucontext->uc_stack.ss_sp;

    // A variable within the current stack frame
    char stackVariable;

    // On Linux, the stack grows down towards smaller addresses
    size_t stackRemaining = &stackVariable - reinterpret_cast<char*>(stackLocation);

    std::cout << "On signal stack at " << stackLocation << " with " << stackRemaining << " bytes of stack memory remaining" << std::endl;
    
    // Overflow the sigaltstack
    overflowTheStack();
}

int main() {
    int ret;

    constexpr size_t stackSize = 16 * 1024;
    void* stackMemory = malloc(stackSize);
    assert(stackMemory != nullptr);
    stack_t newStack{.ss_sp = stackMemory, .ss_flags = 0, .ss_size = stackSize};
    ret = sigaltstack(&newStack, nullptr);
    assert(ret == 0);

    struct sigaction sigact;
    sigact.sa_sigaction = signalHandler;
    sigact.sa_flags = SA_SIGINFO | SA_ONSTACK | SA_NODEFER;
    ret = sigaction(SIGSEGV, &sigact, nullptr);
    assert(ret == 0);
    std::cout << "Causing a stack overflow" << std::endl;
    overflowTheStack();
    return 0;
}
```
```
Causing a stack overflow
On signal stack at 0xac22b0 with 13159 bytes of stack memory remaining
On signal stack at 0xac22b0 with 13159 bytes of stack memory remaining
On signal stack at 0xac22b0 with 13159 bytes of stack memory remaining
...
```
[godbolt](https://godbolt.org/z/vEaxeevPM)
