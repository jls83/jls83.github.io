---
title: C++ Inherited Properties
categories:
- Programming
date: "2022-02-24"
publishdate: "2022-02-24"
tags:
- c++
- python
- javascript
- inheritance
- polymorphism
---

### Inheritance In Python & JavaScript
So here's a fun thing that bit me on my C++ learning journey. In my previous experience with Python, it was (relatively) common to set a property (or properties) on a base class, then make use of that property within the methods of that base class.
```python
class Base:
    def __init__(self):
        self.some_int = 0

    def get_int(self):
        return self.some_int
```

For derived classes, we could simply override the property.
```python
class Derived(Base):
    def __init__(self):
        self.some_int = 1
```

And the methods would use the "closest" `some_int` property in their results.
```python
if __name__ == "__main__":
    foo = Base()
    bar = Derived()

    print("From foo:", foo.get_int())
    print("From bar:", bar.get_int())

    # Prints:
    # From foo: 0
    # From bar: 1
```

In ES6-flavored JavaScript, things work in a similar way.
```javascript
class Base {
    constructor() {
        this.some_num = 0;
    }

    get_num() {
        return this.some_num;
    }
}

class Derived extends Base {
    constructor() {
        super();
        this.some_num = 1;
    }
}

const foo = new Base();
const bar = new Derived();

console.log("From foo:", foo.get_num());
console.log("From bar:", bar.get_num());

// Prints:
// From foo: 0
// From bar: 1
```

### Inheritance In C++ - First Attempt
In C++ things are a bit different. Let's start with our base class one more time.
```cpp
class Base {
    public:
        Base() : some_int_(0) {}

        int getInt() { return some_int_; }
    private:
        int some_int_;
};
```

I've tried my best to write "idiomatic" C++ code here, including the use of a private variable with a public getter. That aside, this should look pretty similar to the base classes from the Python & JavaScript examples.

Now, let's construct our derived class.
```cpp
class Derived : public Base {
    public:
        Derived() : some_int_(1) {}
    private:
        int some_int_;
};
```

Note here that we have to re-declare `some_int_` within `Derived`, or else we get an error like the following:
```
main.cc:15:21: error: member initializer 'some_int_' does not name a non-static data member or base class
        Derived() : some_int_(1) {}
```

Finally, we put the whole thing together & run the program.
```cpp
#include <stdio.h>

class Base {
    public:
        Base() : some_int_(0) {}
        int getInt() { return some_int_; }
    private:
        int some_int_;
};

class Derived : public Base {
    public:
        Derived() : some_int_(1) {}
    private:
        int some_int_;
};

int main() {
    Base foo = Base();
    Derived bar = Derived();

    printf("From foo: %d\n", foo.getInt());
    printf("From bar: %d\n", bar.getInt());
    return 0;

    // Prints:
    // From foo: 0
    // From bar: 0
}
```

Horsefeathers! Unfortunately, C++ diverges from Python & JavaScript on this pattern. When we call `Derived::getInt()`, we're _actually_ calling `Base::getInt()`, as the `getInt` method is not defined for the `Derived` class. Additionally if `Derived` inherits from `Base`, and they both define a member named `some_int_`, _both of the variables exist independently at the same time_ (credit to [this Stack Overflow post](https://stackoverflow.com/a/23776250) for the language there). This is in direct opposition to the way Python & JavaScript operate, where defining a member in a derived class essentially erases the base class's definition of that member.

### Inheritance In C++ - Second Attempt
To make this sort of pattern work, we also have to re-declare the `getInt` method on the `Derived` class.
```cpp
class Derived : public Base {
    public:
        Derived() : some_int_(1) {}
        int getInt() { return some_int_; } // <<<<
    private:
        int some_int_;

    // SNIP

    // Prints:
    // From foo: 0
    // From bar: 1
```

This forces `Derived::getInt()` to look "locally" for the `some_int_` variable, which then returns the right value.

However! This doesn't feel quite right to me. Aside from the duplication of code, it's perhaps a bit too-contrived of an example.

### Inheritance In C++ - Third Attempt
Let's revisit the idea of providing an explicit, parameterized constructor for the `Base` class. We'll include `some_int` as an argument to this constructor.
```cpp
class Base {
    public:
        Base() : some_int_(0) {}
        Base(int some_int) : some_int_(some_int) {}
        int getInt() { return some_int_; }
    private:
        int some_int_;
};
```

Note that we keep the zero-argument constructor for `Base` in place. This polymorphism-first pattern is (to me) one of the distinguishing characteristics of C++ versus the languages I'm more familiar with. After these changes we'll call this new single-argument `Base` constructor within the zero-argument `Derived` constructor. We can also remove the re-declarations of both `getInt` _and_ the `some_int_` member variable.
```cpp
class Derived : public Base {
    public:
        Derived() : Base(1) {}
};
```

Then, we put the whole thing together & run the program.
```cpp
#include <stdio.h>

class Base {
    public:
        Base() : some_int_(0) {}
        Base(int some_int) : some_int_(some_int) {}
        int getInt() { return some_int_; }
    private:
        int some_int_;
};

class Derived : public Base {
    public:
        Derived() : Base(1) {}
};

int main() {
    Base foo = Base();
    Derived bar = Derived();

    printf("From foo: %d\n", foo.getInt());
    printf("From bar: %d\n", bar.getInt());
    return 0;

    // Prints:
    // From foo: 0
    // From bar: 0
}
```

Ahh, now that's more like it! Though we're still actually calling `Base::getInt()` when calling `bar.getInt()` in the example above, the underlying value is set appropriately because of the way we've written the constructors. The overall shape of the code ends up looking pretty darn similar, just with a subtle difference in where we do the variable "override" thanks to the polymorphic constructor on the `Base` class.
