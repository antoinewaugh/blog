# Pure .. Virtual Destructors

Turns out that pure virtual destructors require an implementation.

Whilst I've surely read that tidbit, it wasn't immediatley apparent in the following situation:

```
//state.h
struct State {
    virtual ~State() = 0;    
};

//plugin.cpp
struct Plugin: public State {
    ~Plugin() {
        
    }
};
```

So here our plugin is publically inheriting from our base class, `State`. As state had a pure virtual destructor we ensured that our derived class `Plugin` also had a destructor.

Compile, run and `BOOM`.

`Undefined Symbol: State::~State()`;

Huh? It's meant to be an interface how can we have an undefined symbol to a pure virtual function which _is not meant to have a definition_?.

## nm to the rescue:

```
$ nm mylib.so | c++filt | grep State

U State::~State();

```

Okay fair enough, the lib we have built has an undefined reference itself to the very virtual destructor in question, despite the fact that it is the project containing the header file of the `State` interface.

## virtual destructors are a special case **

Turns out it destructors are a special case: It stems from the fact that after a derived destructor is called, the base destructor is necessarily called _even if it is pure virtual_.

```
#include <stdio.h>

struct Base {
    virtual ~Base() = 0; // pure virtual
};

Base::~Base() {
    printf("~Base");
}

struct Derived: public Base {
    ~Derived() {
        printf("~Derived");
    }
};

int main() {
    Derived d;
}

// Output: ~Derived()~Base()
```

*So next time* you are writing a base class with a pure virtual destructor, please for heavens sake include a definition :/



