# ABI -> I'll learn about that thing later...

## The problem:

It started out so innocently with a shared library consisting of one function (`DoWork`), a struct (`WorkParams`) containing the parameters to pass to the `DoWork` function, and a callsite executable which linked against the shared library containing the DoWork impl.

The work library was to be a shared object (as opposed to header only) such that the implementation of the DoWork could be hidden from the callsite user(s).

The `libwork` was to be compiled on CentOS7, the `myexec` was to be compiled on Fedora, both with gcc version 7+.

So what could possibly go wrong....

Lets start with the shared library and executable code itself:

### Shared Library (CentOS7)
```
// work.h 
typedef struct {
	long timestamp; 
	int arrayOne[10]; 
	int arrayTwo[10]; 
} WorkParams;

bool DoWork(struct WorkParams);

// work.cpp 
bool DoWork(struct WorkParams) {
	// sample impl
	return arrayOne[0] > arrayTwo[0];	
}
```

### Callsite Executable (Fedora27)
```
#include work.h
#include <iostream>

int main() {
	WorkParams params;
	params.timestamp = 1234;
	params.arrayOne[0] = 10;
	params.arrayTwo[0] = 16;

	auto result = DoWork(params);
	std:cout << result;
}

// Output as expected : true
```

So far so good, we had successfully implemented the POC and the callsite binary was outputting valid results for a range of inputs tested. 

## The API Change

There was an additional requirement to store a std::string version of the timestamp named source_timestamp. This was to handle occurences where a string version of the timestamp could hold precision greater than what can be contained in a long.

The structure then took the following form:

```
#include <string>

typedef struct {
	std::string source_timestamp; // recent addition
	long timestamp; 
	int arrayOne[10]; 
	int arrayTwo[10]; 
} WorkParams;

```

### Havoc

After the modification to the header file both the shared object and callsite executable were recompiled, and compilation and linking occurred without any warnings or errors.

What followed was hours of investigation into why `DoWork` was returning inconsistent results.

### Diagnosing

As the latest change was the addition of std::string to the struct, the first response was to remove std::string from the struct altogether. Re-compiling, linking and running without the std::string yeilded correct results. Conclusion: std::string was causing an issue with the integrity of the `DoWork` function.

Re-adding the std::string, but this time to the bottom of the struct, also yielded correct results:

```
typedef struct {
	long timestamp; 
	int arrayOne[10]; 
	int arrayTwo[10]; 
	std::string source_timestamp;
} WorkParams;

```

This did not fill me with confidence. We now had a scenario `std::string` was introducing unexpected behaviour when calling `DoWork`, but only in the instance that it was the first attribute in the struct.

Furthering this, when printing the values of arrayOne[0] and arrayTwo[0] from within `DoWork` yielded different results depending on the location of where `std::string source_timestamp` was placed within the struct.

This suggested that the size of `std::string` and that of `WorkParams` may differ between the library and executable.

#### -fPIC

Position independent code was something I'd been exposed to in the past. It had been a required flag when compiling static libraries that were going to be used by a shared library. Reading into PIC it made sense that if there was expectations/references to memory locations in the shared library that these could not be absolute but needed to be relative.

After speaking with the library author, however, they were already adding the `-fPIC` flag to the shared object creation, and adding it to the g++ -o command did not result in any improvements to the output of `DoWork`.

#### Different standard library versions

I figured that a change in the standard library could have an impact. 

Using `readelf`, the output of the libwork.so and myexec appeared to be using consistent versions of GLIBCXX

```
$ readelf -a libmylib.so

Version needs section '.gnu.version_r' contains 2 entries:
 Addr: 0x0000000000000570  Offset: 0x000570  Link: 4 (.dynstr)
  000000: Version: 1  File: libc.so.6  Cnt: 1
  0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 3
  0x0020: Version: 1  File: libstdc++.so.6  Cnt: 1
  0x0030:   Name: GLIBCXX_3.4  Flags: none  Version: 2```
(edited)

```

```
$ readelf -a myexec

Version needs section '.gnu.version_r' contains 4 entries:
 Addr: 0x0000000000401d00  Offset: 0x001d00  Link: 6 (.dynstr)
  000000: Version: 1  File: libgcc_s.so.1  Cnt: 1
  0x0010:   Name: GCC_3.0  Flags: none  Version: 15
  0x0020: Version: 1  File: libpthread.so.0  Cnt: 1
  0x0030:   Name: GLIBC_2.2.5  Flags: none  Version: 4
  0x0040: Version: 1  File: libc.so.6  Cnt: 2
  0x0050:   Name: GLIBC_2.14  Flags: none  Version: 12
  0x0060:   Name: GLIBC_2.2.5  Flags: none  Version: 3
  0x0070: Version: 1  File: libstdc++.so.6  Cnt: 10
  0x0080:   Name: GLIBCXX_3.4.13  Flags: none  Version: 14
  0x0090:   Name: GLIBCXX_3.4.21  Flags: none  Version: 13
  0x00a0:   Name: CXXABI_1.3.2  Flags: none  Version: 11
  0x00b0:   Name: GLIBCXX_3.4.14  Flags: none  Version: 10
  0x00c0:   Name: GLIBCXX_3.4.5  Flags: none  Version: 9
  0x00d0:   Name: GLIBCXX_3.4.11  Flags: none  Version: 8
  0x00e0:   Name: CXXABI_1.3  Flags: none  Version: 7
  0x00f0:   Name: CXXABI_1.3.3  Flags: none  Version: 6
  0x0100:   Name: GLIBCXX_3.4.22  Flags: none  Version: 5
  0x0110:   Name: GLIBCXX_3.4  Flags: none  Version: 2
```

ldd gave a similar answer, suggesting the libstdc++ shared object was the same: libstdc++.so.6

```
$ ldd ../../lib/libcme_hft.so 

ldd: warning: you do not have execution permission for `../../lib/libcme_hft.so'
    linux-vdso.so.1 (0x00007ffe3dd7b000)
    libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007fabc41c4000)
    libm.so.6 => /lib64/libm.so.6 (0x00007fabc3e6f000)
    libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fabc3c58000)
    libc.so.6 => /lib64/libc.so.6 (0x00007fabc3875000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fabc474e000)```

$ ldd ../../cmake-build-debug/signal_outcome

    linux-vdso.so.1 (0x00007ffe14bc4000)
    libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fb8d9ecf000)
    libcme_hft.so => /home/antoine/CLionProjects/signal_outcome/lib/libcme_hft.so (0x00007fb8d9ccc000)
    libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007fb8d9945000)
    libm.so.6 => /lib64/libm.so.6 (0x00007fb8d95f0000)
    libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fb8d93d9000)
    libc.so.6 => /lib64/libc.so.6 (0x00007fb8d8ff6000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fb8da0ee000)
```

#### sizeof

Irrespective of `readelf` and `ldd` suggesting stdlibc++ was consistent between `libwork` and `myexec` on the different machines, the relationship between `DoWork` providing unexpected results being dependent on the location of the `std::string source_timestamp` within the `WorkParams` struct still pointed to an alignmnet difference.

Printing the `sizeof(std::string)` and `sizeof(WorkParams)` from both within `libwork` and `myexec` yielded interesting results:

```
sizeof(WorkParams) = 200, sizeof(std::string): 8  // libwork output 8
sizeof(WorkParams) = 224, sizeof(std::string): 32 // myexec output 32
```
 
Aha! We now have an explination of WHAT is causing our issue. Yet we are still missing the WHY.

### The Slack family

Thankfully - after asking what could be causing a sizeof(std::string) to differ between 8 and 32 bytes across Centos/Fedora builds, tit was suggested by @toby_allsop that c++11 `std::string` contained 3 or more pointers (8 bytes each), whereas `std::string` was just a single pointer (8 bytes) in pre c++11 implementations.

A suggestion from @toby_allsop and @grisumbras was that differing *ABIs* could be causing the problem.

Once GCC version was confirmed to be version was 7.2.1 on CentOS, and 7.3.1 on Fedora this seemed less plausible as both are expected to default to the same ABI (1). Documentation supported this https://gcc.gnu.org/onlinedocs/libstdc++/manual/abi.html 

Even if the ABI was different, questions such as how could this be the case between such close versions of gcc as well as how mywork.so could be using a non-c++11 std::string given the -std=c++11 flag had been provided when compiling the library were still outstanding.

#### Macros to the rescue

Using a GCC macro, it is possible to see the default ABI settings for a given gcc toolchain:

```
//abi.cpp
#include <iostream>
int main() {
	std::cout << _GLIBCXX_USE_CXX11_ABI;
}
```

Running the above codebase on both CentOS and Fedora:

```
// CentOS
$ g++ abi.cpp 
$ a./out
0

// Fedora
$ g++ abi.cpp 
$ a./out
1
```

This was enlightening indeed. It suggested that although gcc versions were nearly identical, it was plausible that `std::string` was being represented differently between the two systems. This explained discrepancy in `sizeof` output.

*In an attempt to compile CentOS with a flag indicating CXX11_ABI should be 1, the output still remained at 0*.

```
// CentOS
$ g++ -D_GLIBCXX_USE_CXX11_ABI=1 abi.cpp
$ ./a.out
0
```

Hmmmmm.... WTF.

#### Reading up on ABI

I was pointed to a key link in which the gcc manual states from version 5.1 a dual ABI is supported. https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html

This change was made to allow for backwards compatibility allowing c++11 features to be linked with pre c++11 codebases (c++03 for example.).

The `libstdc++.so` was NOT renamed, and therefore the checks on readelf and ldd were not sufficient to determine if the same standard std::string would be used, because it effectively offers both pre and post c++ std::strings! Newer std::strings have been placed under the namespace std::__cxx11::string such they may co-exist, at least at the ABI level.

Furthering to the confusion, whilst _GLIBCXX_USE_CXX11_ABI defaults to 1 from gcc version 5 onwards: 
_*some linux distributions configure gcc differently such that the default value is still 0*_

Urgh.

In addition - the ABI itself and the c++ standard specified in the compilation _are indepdent_. In other words: the `-std=c++11` flag does not effect the ABI and you can/may/will still end up with ABI of std::string pointing to the legacy implementation if the _GLIBCXX_USE_CXX11_ABI defaults/is set to 0.

CentOS also ignored attempts to override the default, and as the gcc manual suggests"

_"If the third-party library cannot be rebuilt with the new ABI then you will need to recompile your code with the old ABI."_

This implies that rather than CentOS being compiled with ABI=1, I could compile the Fedora myexec using ABI=0.

#### Success! 

```
// Fedora
$ g++ -DGLIBCXX_USE_CXX11_ABI=0 abi.cpp
$ ./a.out

0
```

After confirming that Fedora complied with a matching ABI to CentOS, I was able to link `myexec` to `libwork.so` on Fedora , with the `source_timestamp` located at the top of the `WorkParams` and the data flow worked perfectly.

### Lessons Learnt

This was a painful lession in which I learnt:

* ABI and c++ dialect are independent
* different gcc versions may differ in ABI
* same gcc versions, on different distributions may differ in ABI
* specifying `-std=c++11` does not enforce std::string to be taken from the c++11 library on gcc distributions where ABI defaults to 0
* distributions where ABI defaults to zero may be hard-coded to ignore the macro setting when passed to gcc

It has become apparent that confirming the ABI in use is CRITICAL before linking to any `*.so` to avoid such pains in the future.

