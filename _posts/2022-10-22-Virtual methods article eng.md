---
layout: default
title:  "Untangling Virtual Methods"
date:   2022-10-22 16:11:16 +0300
---

![]({{ site.baseurl }}/assets/images/virtual/madmark.png)

I'm one of those weird guys who from time to time write answers on [r/systemverilog](https://www.reddit.com/r/systemverilog/) One day I saw there a rather simple question from my point of view: what is a virtual method? A protracted discussion showed that the question isn't that simple if you come to SystemVerilog with no OOP experience in other languages. Inheritance, method overrides, reference types - all these are the concepts crucial to understanding virtual methods work.

Let's try to figure it out.

Referring to the language standard, we will mean [1800-2017 - IEEE Standard for SystemVerilog](https://ieeexplore.ieee.org/document/8299595). The code will be tested in four simulators: Riviera, VCS, Xcelium, Questa, but EDA Playground without a corporate account is enough to reproduce all examples.

# Why do we need virtual methods?
Imagine a driver that receives packets and sends them over some interface as a bitstream. Packets can be of types `data` and `control`, each type having its own set of fields that characterize the packet. It would be nice to write the code of the driver to handle both `data` and `control` at once, and possibly keep this code unchanged in the future, when the packet types will multiply faster than problems in 2022. Such code could look like this:

```SystemVerilog
class driver;
  task drive_pkt(packet pkt);
    int raw = pkt.to_raw();
    foreach(raw[i]) begin
      // drive bit
    end
  endtask
endclass
```
The `drive_pkt` method can take any class derived from `packet` as an argument. Each subclass has its own implementation of the `to_raw` method. When adding new package types, one only need to implement this method in a new class, and there is no need to alter the driver code.

For this scheme to work, the `to_raw` method must be virtual.

# How virtual methods work
## The direct approach
Let's start with the definition from the standard.
>A method of a class may be identified with the keyword virtual. Virtual methods are a basic polymorphic construct. A virtual method shall override a method in all of its base classes, whereas a non-virtual method shall only override a method in that class and its descendants. One way to view this is that there is only one implementation of a virtual method per class hierarchy, and it is always the one in the latest derived class.

Is it clear for you? Definitely not for me. But let's take a closer look at the last sentence, it doesn't look scary.
>One way to view this is that there is only one implementation of a virtual method per class hierarchy, and it is always the one in the latest derived class.

So, the virtual method of the derived class overrides the virtual method of the parent class, and what object's method we call, we will get the implementation of the last derived class? [Let's check.](https://www.edaplayground.com/x/khQD).
```SystemVerilog
class Foo;
  function void my_common_name();
    $display("Foo");
  endfunction
  
  virtual function void my_virtual_name();
    $display("Foo");
  endfunction
endclass

class Bar extends Foo;
  function void my_common_name();
    $display("Bar");
  endfunction
  
  virtual function void my_virtual_name();
    $display("Bar");
  endfunction
endclass

module tb;
  initial begin
    Foo foo = new();
    Bar bar = new();
    foo.my_virtual_name();
    bar.my_virtual_name();
  end
endmodule
```
Output:
```
# KERNEL: Foo  
# KERNEL: Bar
```
It turns out that after all, more than one implementation of the `my_virtual_name` method exists. So, is the standard deceiving us?

Not necessarily. It's just the explanation in the standard is written in such a fabulous way that it can hint at the real behaviour of virtual methods only to those who already understand it. So let's throw it out it give our own explanation.
>When a virtual method is called, the implementation is chosen not by the handle type, but by the type of the object the handle points to.

If you still don't understand, don't worry. We will get into it in a minute. But first - an example demonstrating the "virtuality" of methods. Let's take the code from the previous example and change the `initial` block.
```SystemVerilog
  initial begin
    Foo foo;
    Bar bar = new();
    foo = bar;
    foo.my_virtual_name();
    foo.my_common_name();
  end
```
Output:
```
# KERNEL: Bar
# KERNEL: Foo
```
Although the handle is of type `Foo`, it points to an object of type `Bar` whose `my_virtual_name()` method was called, while the non-virtual `my_common_name()` method was called according to the type of the handle.

## The indirect approach
It is crucial to understand the difference between reference types and value types.

When declaring an integer variable
```SystemVerilog
int x = 4;
```
the variable `x` is the number 4. We can assign a different value to it, then this variable will become a different number. The assignment operator in this case can be thought of as writing the number on the right into the memory location on the left.

It's more complicated when declaring an object.
```SystemVerilog
Foo foo = new();
```
Here `foo` is not an object, but a handle (can also be named "reference", the distinction is not important for us). An object of type `Foo` is somewhere in memory, and a handle `foo` points to that location in memory. The type of a handle does not have to be the same type  as the object it points to. It can also be an object of any type that is lower in the class hierarchy. That is, any derived class.
```SystemVerilog
foo foo;
Bar bar = new();
foo=bar;
```
It doesn't work in reverse. A handle cannot point to an object higher in the class hierarchy than the handle type.
```SystemVerilog
Foo foo = new();
Bar bar;
bar = foo; // compilation error
```
We can draw the following analogy: a handle is a box, and an object is placed in this box by an assignment operator. The derived class object is smaller, and its box is also smaller. In the example above, `foo = bar` means "take an object from the box `bar` and put it into the box `foo`". It is impossible to execute `bar = foo` because the box `foo` currently contains an object too large to fit into `bar`.

What is the difference between virtual methods and non-virtual methods in our analogy? When a non-virtual method is called, the simulator only looks at the box, and when a virtual method is called, it looks at the contents of the box.
![]({{ site.baseurl }}/assets/images/virtual/box-g3c0edfd68_640.jpg)

### More details less boxes
A brief explanation for those who are interested in what is happening under the hood.

Each time a function is called, the simulator needs to know at what address in memory the function code is located. If the method is non-virtual, then this information is known at compile time, and wherever the method is called in the source code, a jump to the memory address with this method appears in the executable code. This is called *early binding*.

If the method is virtual, then for early binding, the compiler needs to know the type of the object the handle points to, which is rarely possible. Therefore, for each class with virtual methods, a table of virtual functions (*vtable*) is created, containing pairs "index - address of the method". A pointer to this table (*vpointer*) is added to the class as a member. When simulator has to call a virtual method, it finds vtable using vpointer and then finds the method address using the method's name as a table index.

I'm not entirely sure that this is how virtual methods work in SystemVerilog, but that's how they work in C++ and similar languages. Given how similar SystemVerillog is to C++, it is safe to assume that this explanation is relevant.

## More complicated cases
![]({{ site.baseurl }}/assets/images/virtual/bill.png)

### Calling a method within an object
I hope that virtual methods does not present a puzzle anymore. But so far we've been calling virtual methods using handles. What about calling virtual methods within the object itself?

In the following example, we will call a virtual function inside a non-virtual one. Recall, that there's an optional handle `this`, which can be used to access object's members from within. So we should not expect to see a different behaviour, right? [Let's check](https://edaplayground.com/x/Uthi).
```SystemVerilog
virtual class foo;
  virtual function void my_name();
    $display("foo");
  endfunction
  
  function new();
    my_name();
  endfunction
  
  function wen();
    my_name();
  endfunction
endclass
    
class bar extends foo;
  virtual function void my_name();
    $display("bar");
  endfunction
  
  function new();
    super.new();
  endfunction 
  
  function wen();
    super.wen();
  endfunction
endclass
    
module top;
  initial begin
    automatic bar x = new();
    x.wen();
  end
endmodule
```

Which simulator behaves according to your expectation?

| Simulator | Output |
| --------- | ------ |
| VCS       | bar<br>bar       |
| Questa    | foo<br>bar       |
| Xcelium   | bar<br>bar       |
| Riviera   | foo<br>bar       |

Calling `x.wen()` in all simulators printed `bar` as expected. But what happened in the constructor?

According to the virtual methods rules, the `my_name()` method from the `bar` class should have been called. However, at the time of the call, the execution was inside the `foo` constructor, while the `bar` object's construction had not even begun had not even begun, making the situation ambiguous .

If you think VCS and Xcelium are just a little smarter, I'll have to disappoint you. This behaviour is dangerous. Let's add another class.
```SystemVerilog
class baz;
  int x = 1;
endclass
```
And change `bar`.
```SystemVerilog
class bar extends foo;
  baz baz_o = new();
  
  virtual function void my_name();
    $display("bar");
    $display(baz_o.x);
  endfunction
```
Full code: https://edaplayground.com/x/MFhz

What happens now? According to paragraph 8.7 of the standard, the construction of an object occurs in the following order:
1. The class constructor calls the base class constructor.
2. After the base class constructor completes, the class fields are initialised.
3. After that, the code of the class constructor is further executed.
```SystemVerilog
class bar extends foo;
  baz baz_o = new();
  functionnew();
    super.new();
    // baz_o = new() executes here
  end function
```
That is, at the time of the call to `super.new()`, the `baz_o` field has not yet been initialised and is equal to `null`. If you run the simulation in VCS, it will crash with a Null Object Access error. The same fate awaits Xcelium. Riviera and Questa feel great because they don't try to access someone else's variable.

In C++, the virtual method mechanism is [disabled in the constructor](https://stackoverflow.com/questions/962132/calling-virtual-functions-inside-constructors) to protect against just such a danger. Whether the developers of Riviera and Questa follow this practice consciously, or by chance, I don’t know.

So which simulator is correct? The one followed the standard literally, or the one didn't get shot in the foot? We can certainly say that the bad guys are
1. the standard, that does not highlight the case when virtual methods simply cannot work as they should;
2. the developer who avoided thinking too much.

In other words, just don't call a virtual method in a constructor.

But what to do if you really-really need to call a virtual method in a constructor? Sometimes it's necessary at the object's initialisation. Since a virtual method cannot be used in a constructor, we need to move it out. The only question is where. Let's consider two possible options.

We can call the necessary virtual method whenever an object is created.
```SystemVerilog
bar x = new();
x.my_name();
```
An extremely simple solution, but not very safe: it's too easy to forget that you need to perform an additional action after the construction.

It will be safer to make the constructor `protected`, add a static method to the class and create an object using it..
```SystemVerilog
class bar extends foo;
  static function bar create();
    bar x = new();
    x.my_name();
  end function

  protected function new();
    // ...
  end function
end class
```
Now we can't create an object without calling to `my_name()`. The disadvantage of this approach is that such a class cannot be used with a UVM factory.

Which method to choose, you have to decide according to the circumstances. There are other solution to this problems, of course, but we will stop here as this is a topic for another time.

### Change method's signature
When a method is overridden, its input arguments and return type need not be exactly the same as its parent. Below is a slightly modified example from clause 8.20 of the standard.
```SystemVerilog
typedef int T; // T and int are matching data types.
typedef bit signed [31:0] MyInt;

class Foo; endclass
class Bar extends Foo; endclass

class C;
	virtual function Foo some_method(int a); endfunction
endclass

class D extends C;
	virtual function Bar some_method(T a); endfunction
endclass

class E #(type Y = logic) extends C;
	virtual function Bar some_method(Y a); endfunction
endclass

module tb;
  initial begin
    // E#() ee = new(); Error: logic and int are not matching types
    E#(MyInt) e = new();
  end
endmodule
```
If the return type is a class, it's okay to replace it with a derived class. If it's not, then it can be replaced with a matching type. Roughly speaking, the matching type is the same type under a different name (compare `int` and `MyInt`), but there are nuances with structures. More about this in paragraph 6.22.1 of the standard.

The type of the input arguments can be replaced with a matching type.

It is also allowed to add or omit the `virtual` keyword in derived classes.
https://edaplayground.com/x/GZbg
```SystemVerilog
class A;
  function void my_name();
    $display("A");
  endfunction
endclass

class B extends A;
  virtual function void my_name();
    $display("B");
  endfunction
endclass

class C extends B;
  function void my_name();
    $display("C");
  endfunction
endclass

class D extends C;
  function void my_name();
    $display("D");
  endfunction
endclass

module automatic tb;
  initial begin
    A a = new();
    B b = new();
    C c = new();
    D d = new();
    a = b;
    a.my_name();
    c = d;
    c.my_name();
  end
endmodule

```
In class `A` the method was not virtual and will behave as non-virtual when a handle of type `A` is used. However, once it is overridden as virtual in class `B`, it will become virtual in all subclasses, regardless of whether the `virtual` modifier is present.

Output:
```
# KERNEL: A  
# KERNEL: D
```
From a code smell point of view, it is best not to omit `virtual`: do not force others to look through the whole hierarchy looking for a single word. Exceptions to this are maybe some obvious cases like UVM phases. Everyone knows that they are virtual.

### Calling a specific method implementation
Despite the assurances of the standard that there is only one implementation of a method, and the virtual method rules, there is a case where we can call a base class' implementation.

https://edaplayground.com/x/WbsT
```SystemVerilog
class Foo;
  virtual function void my_name();
    $display("Foo");
  endfunction
endclass

class Bar extends Foo;
  virtual function void my_name();
    super.my_name();
    $display("Bar");
  endfunction
endclass

class Baz extends Bar;
  virtual function void my_name();
    super.my_name();
    $display("Baz");
  endfunction
endclass

module tb;
  initial begin
    Foo foo;
    Baz baz = new();
    foo = baz;
    foo.my_name();
  end
endmodule
```
In this regard, virtual methods do not differ from non-virtual. 
```
# KERNEL: Foo  
# KERNEL: Bar  
# KERNEL: Baz
```
Sometimes we want to call the `super.super` implementation instead of just `super`. This is the case when the grand-parent class performs some initialisation and we want to fully override the parent's behaviour. Writing `super.super` is not allowed, but the problem is easily solved.
```SystemVerilog
class Baz extends Bar;
  virtual function void my_name();
    Foo::my_name();
    $display("Baz");
  endfunction
endclass
```
All simulators produce the same output:
```
# KERNEL: Foo  
# KERNEL: Baz
```
That is, the `::` operator allows us to call an implementation of a particular class. But in the last example the call was made from the same method, which implementation we wanted to chose. Is it possible to call the implementation of a method outside that method? Let's add a non-virtual function to the `Foo` class and try to call `Foo::my_name`.
```SystemVerilog
class Foo;
  virtual function void my_name();
    $display("Foo");
  endfunction
  
  function try_to_choose();
    Foo::my_name();
  endfunction
endclass

class Bar extends Foo;
  virtual function void my_name();
    super.my_name();
    $display("Bar");
  endfunction
endclass

class Baz extends Bar;
  virtual function void my_name();
    super.my_name();
    $display("Baz");
  endfunction
endclass

module tb;
  initial begin
    Foo foo;
    Baz baz = new();
    foo = baz;
    foo.try_to_choose();
  end
endmodule
```

| Simulator | Output |
| --------- | ------ |
| VCS       | Foo<br>Bar<br>Baz       |
| Questa    | Foo       |
| Xcelium   | Foo       |
| Riviera   | Foo<br>Bar<br>Baz       |

Opinions diverged. Questa and Xcelium called the method of the chosen class, while VCS and Riviera called it as a virtual method. On the one hand, this is closer to the rules of virtual methods. On the other hand this is not what the author intended. However, making a method virtual **and** relying on calling its concrete implementation is not the best idea. We have to choose one or the other.

# Applications
Now that we understand how virtual methods work, let's move on to the question of their application.

## Polymorphism
This is the ability of a function to handle different data types. This is exactly what we wanted in the package processing problem. Let's complete that code. 

```SystemVerilog
virtual class packet;
  pure virtual function int to_raw();
endclass

class data_pkt extends packet;
  virtual function int to_raw();
  // convert to integer
  endfunction
endclass
    
class control_pkt extends packet;
  virtual function int to_raw();
  // convert to integer
  endfunction
endclass
    
class driver;
  task drive_okt(packet pkt);
    int raw = pkt.to_raw();
    foreach(raw[i]) begin
      // drive bit
    end
  endtask
endclass
```

By now it should be clear to you why the `to_raw` method needs to be virtual and how the driver will work. Let us note the details that we have not touched before.

The base `packet` class itself is declared as `virtual`. This means that an object of this class cannot be created, i.e. the following code cannot be compiled.
```SystemVerilog
packet pkt = new();
```
Such a class is called *abstract*. Abstract classes are useful when the class is so incomplete that it makes no sense to create and use its objects. If the packet is always either data or control, then the existence during simulation of a packet that is neither is an error that needs to be handled. Instead, you can declare the class abstract and make such an error impossible.

The `to_raw` method in the `packet` base class is declared as `pure virtual`. This means that the class does not provide an implementation of this method, but its non-abstract subclasses must have an implementation of this method. In other words, the implementation must be provided by the first non-abstract subclass in the hierarchy.

## Change existing behaviour
Sometimes we need to add some features or change the behaviour of existing components. For example, a new out of band signal has been added to a standard interface and shall be supported by the agent. Or we want to mess up valid transactions in the test to check DUT's error handling. In both cases, it is advisable not to copy-paste code that can be reused.

Both tasks can be solved using polymorphism and UVM factory. Let's consider the transaction issue.
https://edaplayground.com/x/aKcd
```SystemVerilog
import uvm_pkg::*;
`include "uvm_macros.svh"

class transaction extends uvm_object;
  `uvm_object_utils(transaction)
  
  rand bit [3:0] body;
  
  function new(string name = "transaction");
    super.new(name);
  endfunction
  
  virtual function bit parity();
    return ^(body);
  endfunction
endclass

class error_transaction extends transaction;
  `uvm_object_utils(error_transaction)
  
  function new(string name = "error_transaction"); endfunction
  
  virtual function bit parity();
    return !super.parity();
  endfunction
endclass

module tb;
  int count = 0;
  always #1 begin
    automatic transaction t = transaction::type_id::create();
    void'(t.randomize());
    $display("Body: %b; Parity: %0d", t.body, t.parity());
    // do something with transaction
    count++;
  end
  
  initial begin
    wait(count == 4);
    $display("Inject error");
    transaction::type_id::set_type_override(error_transaction::get_type());
    wait(count == 6);
    $display("Return to normal");
    transaction::type_id::set_type_override(transaction::get_type(), 1);
    wait(count == 8);
    $finish();
  end
endmodule
```
Here we wait for the four original transactions to be sent, use the factory to replace the type with an error transaction and after two more transactions, we replace the type back.
```
# KERNEL: Body: 0110; Parity: 0  
# KERNEL: Body: 0010; Parity: 1  
# KERNEL: Body: 1101; Parity: 1  
# KERNEL: Body: 0000; Parity: 0  
# KERNEL: Inject error  
# KERNEL: Body: 0011; Parity: 1  
# KERNEL: Body: 0110; Parity: 1  
# KERNEL: Return to normal  
# KERNEL: UVM_WARNING @ 6: reporter [TYPDUP] Original and override type arguments are identical: transaction  
# KERNEL: UVM_INFO @ 6: reporter [TPREGR] Original object type 'transaction' already registered to produce 'error_transaction'. Replacing with override to produce type 'transaction'.  
# KERNEL: Body: 1001; Parity: 0  
# KERNEL: Body: 0000; Parity: 0  
```
The key point is that the code working with transactions remains unchanged. In a real testbench such code may be buried so deep that changing it is not an option at all.

The factory complains a bit when we override the error transaction back to the original one. This is expected for UVM-1.2, but UVM-1.1d will not be able to revert the type back due to a bug in the factory.

The issue with the driver is solved in a similar way: derive a class from the base driver, override the necessary methods, which must be virtual, use the factory to override the type of the driver being created. The original agent code does not need to be changed in this case, it is completely reused. You can override the type in the factory from anywhere. Good places to do this are the base test and the environment.

Besides the factory, there are other ways to solve both problems. Here we are simply illustrating the possibility. Whether it was the best approach or not is a question for another time.

## Interface classes
This is a vast and complex topic that deserves a separate article. Therefore, we will touch on it only briefly.

An interface class is a collection of purely virtual methods. The difference from an abstract class is that an interface class has no properties (only methods), and is not inherited (extended), but *implemented*. Each class can inherit only one class, but implement any number of interface classes.

As with abstract classes, any class that implements an interface must provide implementations of all pure virtual methods of the interface class.

Objects of all classes that implement some interface class can be used as objects of this interface class, which opens up new possibilities for using polymorphism.

See clause 8.26 of the standard for details, and use cases can be seen in [this great article](http://rockeric.com/wp-content/uploads/2018/04/DVCon2016-US-SystemVerilog-Interface-Classes-% E2%80%93-More-Useful-Than-You-Thought.pdf).

## Where to avoid virtual methods
Finally, let's consider the cases where the use of virtual methods will only get in the way.

First, the standard allows non-virtual methods to be used as `wait` and `@` expressions.
```SystemVerilog
wait(obj.method());
@(obj.method())
```
The standard does not explicitly forbid using virtual methods in this way, so I have no complaints about Xcelium, Questa and Riviera, which compile these expressions with virtual methods. Only in VCS the compilation fails. But you can wrap a virtual method in a non-virtual one, then VCS will compile it.

Second, remember the problem of calling a virtual method in a constructor. If you really need to call the method there and suggested workarounds do not solve your problem, then you should not make the method virtual. Even if you know how your simulator behaves, things may change in the next version.
![]({{ site.baseurl }}/assets/images/virtual/grabli.png)

Third, the correctness of the class may depend on the correct implementation of its internal methods. If such a method is virtual, then overriding it in a subclass can break the the base class. I have not seen such cases in practice, but let's try to imagine a couple.

Suppose, there's a class working with pseudo-random sequences and we override the `get_next_element` method in a subclass. Without a deep knowledge in math it's all to easy to degrade the statistical properties of the sequence. The code will continue to work, but the requirements for the quality of the input data will be violated.

If you are developing a VIP for sale, then the question of licensing arises. If the license check is implemented in a class method, then you really don't want someone to redefine it to `return 1`.

## But in Java, everything is virtual...
Virtual methods allow us to use the flexibility of polymorphism, but there seem to be very few situations where methods must be strictly non-virtual. So maybe it's worth declaring **all** methods virtual? This is a great topic for a holiwar, but I myself stick with a positive answer to this question.

In my practice, I have never had to remove the `virtual` modifier, but I had to add it. Well, who in their right mind, I thought, will override this method here? Of course, it was me, half a year later, who had to override this method. And of course I forgot that this method wasn't virtual and had to spend some time debugging, when I could have declared the method as virtual from the beginning.

To make all methods virtual or not is a question that you need to answer yourself, using your experience and the specifics of your testbenches. However, there is one objective counter-argument against the omnivirtuality - performance.

# Experiments
If you've looked into the section on implementers of virtual methods, you should have noticed the extra overhead of calling a virtual method compared to a non-virtual one.

Let's try to find out how bad it is.

```SystemVerilog
class foo;
  virtual function int virtual_add(int a, int b);
    return a + b + 1;
  endfunction
  
  function int nonvirtual_add(int a, int b);
    return a + b + 1;
  endfunction
endclass

class bar extends foo;
  virtual function int virtual_add(int a, int b);
    return a + b + 2;
  endfunction
  
  function int nonvirtual_add(int a, int b);
    return a + b + 2;
  endfunction
endclass

class tester;
  static function void do_test();
    foo handle;
    longint N = 5000000; // iterations
    longint M = 2; // rounds
    if ($urandom_range(1, 0) == 0) begin
      foo t = new();
      handle = t;
      // handle = foo::new(); Xcelium fails to compile this o_o
    end else begin
      bar t = new();
      handle = t;
    end
    $display("Test virtual");
    repeat(M) begin
      $system("echo $(($(date +%s%N)/1000000)) >./st"); // that's how we get milliseconds
      repeat(N) begin
        int c = handle.virtual_add(1, 2); 
      end
      $system("echo $(($(date +%s%N)/1000000)) >./end");
      $system("s=`cat ./st`; e=`cat ./end`; echo `expr $e - $s`");
    end
    
    $display("Test nonvirtual");
    repeat(M) begin
      $system("echo $(($(date +%s%N)/1000000)) >./st");
      repeat(N) begin
        int c = handle.nonvirtual_add(1, 2);
      end
      $system("echo $(($(date +%s%N)/1000000)) >./end");
      $system("s=`cat ./st`; e=`cat ./end`; echo `expr $e - $s`");
    end
  endfunction
endclass

module tb;
  initial begin
    tester::do_test();
  end  
endmodule 
```
A few clarifications right of the bat.

* A sufficiently smart compiler may in some cases notice that at the moment of calling a virtual method, the type of the class is known unambiguously. In this case, late binding can be replaced by early binding. To eliminate such optimisation, we choose a class randomly.
* The class is selected before the start of the experiment. Firstly, if a class is selected before each method call, it is most likely that the virtual method is needed anyway and the comparison looses its sense. Secondly, I don't want to overburden this already big text and leave the further experiments in the reader's capable hands.

At the time of writing, I do not have the ability to run all simulators on the same hardware, so I will only give relative measurement results.

| Simulator | Virtual slower, % |
| --------- | -------------- |
| Riviera   |       7        |
| Questa    |       64       |
| VCS       |       1        |
| Xcelium   |       10       |

Questa shows a very noticeable slowdown, while other simulators perform not much slower. VCS turned out to be so good that it's even suspicious.

How can such high efficiency be explained? We randomised the object only once, which simplifies the task at the hardware level. The code of the desired method can be stored in the cache, and branch prediction quickly selects this method. This can also explain Questa's problems: it worked through virtualisation. Also, during compilation, the virtual function call could be replaced by two [inline](https://en.wikipedia.org/wiki/Inline_function) functions with simple conditional branching. A more detailed study of the internals of simulators is beyond the scope of this article.

Of course, the possibility of optimisations and their effectiveness strongly depend on the context of the virtual method call. On the other hand, in a real testbench, you can expect that even Questa's slowdown won't be noticeable.

So, don't be afraid of virtual methods because of the additional overhead and don't optimise prematurely.

# Conclusion
* A virtual method is a method whose implementation is chosen according to the type of the object, not the type of the handle.
* Virtual methods allow you to add and change the functionality of existing code without interfering with it.
* Virtual methods should not be used in a constructor, in `wait()` and `@()` expressions, or where there is a dependency on a specific method implementation.
* From a virtual method, you can call its previous implementations using the `::` operator and `super`.
* It takes more time to call a virtual method than to call a non-virtual one. However, this difference is not so great to care about it when creating a method.
