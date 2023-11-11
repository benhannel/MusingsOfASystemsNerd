# Some oddities of sigaction and SA_ONSTACK

Install a signal handler

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

Overflow the stack
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

Install an alternate signal stack
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

Segfault in the signal handler
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

SA_NODEFER
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
