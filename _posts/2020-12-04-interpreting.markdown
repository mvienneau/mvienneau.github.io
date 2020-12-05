---
layout: post
title:  "Interpreting"
date:   2020-12-04 16:50:44 -0700
categories: jekyll update
---
### Writing a Brainfuck Intepreter
#### Part 1: Python
[Brainfuck][brainfuck-wiki] is an esoteric language that is distinguished by it's simple yet, unreadable syntax. As an example, here is `Hello World` in Brainfuck: 

`>+++++++++[<++++++++>-]<.>+++++++[<++++>-]<+.+++++++..+++.[-]>++++++++[<++++>-] <.>+++++++++++[<++++++++>-]<-.--------.+++.------.--------.[-]>++++++++[<+++>- ]<+.[-]++++++++++.`

It's a great language to write an interpreter for, as it only consists of 8 commands. Conceptually, the language operates along a tape incrementing and decrementing the data pointer, as well as the data pointed to by that pointer. In python, parsing those characters looks like the following:

{% highlight python %}
def evaluate(str):
    # str being some brainfuck program (">>>++<+>>>----.")
    # Initialize a class to hold all of our state
    tape = Tape()
    # instruction pointer (where we are at in the program / string)
    i = 0
    while i < len(str):
        char = str[i]
        if char == "+": tape.incr()
        if char == "-": tape.decr()
        if char == ">": tape.movefoward()
        if char == "<": tape.movebbackward()
        if char == ".": tape.output()
        i = i + 1
{% endhighlight %}

This example is missing 3 inputs still, `[`, `]`, and `,`. For this post, I am going to ignore the `,` input and just focus on the brackets -- `[` and `]`.
The brackets are the trickiest part of this, but still relatively simple. They essentially exist to allow for looping functionality within the language. From wikipedia;

`[`: if the byte at the data pointer is zero, then instead of moving the instruction pointer forward to the next command, jump it forward to the command after the matching ] command. 

`]`: if the byte at the data pointer is nonzero, then instead of moving the instruction pointer forward to the next command, jump it back to the command after the matching [ command. 

So, `[` behaves somewhat like
{% highlight python %}
while not (x == 0):
{% endhighlight %}

To break this down further, consider the following block of brainfuck:
{% highlight python %}
>+++++++++[<++++++++>-]
{% endhighlight %}
Imagine the tape looks something like below
{% highlight python %}
0 0 0 0 0 0 ...
^
{% endhighlight %}

The first instruction is `>` which moves the cell/tape/memory pointer to the right, or, increments it and then adds 9 (`++++++++`) to the value
{% highlight python %}
0 9 0 0 0 0 ...
  ^
{% endhighlight %}

Then, we enter the loop via `[` because the memory pointer is pointing at a value that is non zero. Continuing on, we decrement the memory pointer, and increment the value pointed at by 8 (`++++++++`). 
{% highlight python %}
8 9 0 0 0 0 ...
^
{% endhighlight %}

After, increment the pointer again, and decremenet the value in that cell.
{% highlight python %}
8 8 0 0 0 0 ...
  ^
{% endhighlight %}
Now that we have reached the `]` character, and the pointer is pointing at a non-zero value, return to the first character after the matching `[`, or `<`. Following the loop...
{% highlight python %}
8 8 0 0 0 0 ...
^
{% endhighlight %}
{% highlight python %}
16 7 0 0 0 0 ...
^
{% endhighlight %}
Following this all the way through, we see that this is a loop to essentially perform `8 * 9`. Adding 8 to the first cell 9 times. This is also the first operation of printing `Hello World`, as `8*9=72` and `72` is the ASCII decimal value for the letter `H`. 


Now that loops are more understood, the "tricky" part of their implementation in a Brainfuck interpreter is the act of finding the matching brackets. Looking back at the definition, it states "jump it forward to the command after the matching ] command". This translates to the interpreter having to know where to jump when it encounters any bracket, or, storing a list of brackets with their indexes in the program, and their matching bracket index. 

This can be accomplished through some preprocessing of the input program like so:
{% highlight python %}
def find_matching_paren(str):
    se = {}
    stack = []
    # Get all the characters and their indexes
    for i, char in enumerate(str):
        if char == "[":
            # Add this bracket index to the stack
            stack.append(i)
        if char == "]":
            # Get the most recent (matching) "["
            match = stack.pop()
            # store {index_of_bracket: (index_of_bracket, index_of_matching_bracket)}
            se[match] = (match, i)
            se[i] = (match, i)
    return se
{% endhighlight %}

What this code block does, is use a stack to keep track of the matching brackets and their indexes. Walking through this for a program like "[*****]":
{% highlight python %}
i = 0; bracket found;  stack = (0)
i = 1-5; stack = (0) # no encountered brackets
i = 6; bracket found; ]; stack = (); se = {0: (0, 6), 6: (0, 6)}
{% endhighlight %}
`se` here stores the index of a bracket, and the value is a tuple of `(index_of_[, index_of_matching_])`

Putting this all together, the full eval function now looks more like this:
{% highlight python %}
def evaluate(str):
    # str being some brainfuck program (">>>++<+>>>----.")
    # Initialize a class to hold all of our state
    tape = Tape()
    # instruction pointer (where we are at in the program / string)
    i = 0
    while i < len(str):
        char = str[i]
        if char == "+": tape.incr()
        if char == "-": tape.decr()
        if char == ">": tape.movefoward()
        if char == "<": tape.movebbackward()
        if char == ".": tape.output()
        if char == "[":
            if not(tape.check_loop()):
                i = parens[i][1]
        if char == "]":
            if tape.check_loop():
                i = parens[i][0]
        i = i + 1
{% endhighlight %}
Where `check_loop` checks to see if we are currently looping (the value of the data pointer being zero or not)

The implementation of the `Tape` class is straightforward, just a container to keep track of all the state
{% highlight python %}
class Tape():
    def __init__(self):
        self.tape = [0]
        self.pointer = 0
    def incr(self):
        self.tape[self.pointer] += 1
    def decr(self):
        self.tape[self.pointer] += -1
    def movef(self):
        if self.pointer == len(self.tape) - 1: self.tape.append(0)
        self.pointer += 1
    def moveb(self):
        self.pointer += -1
    def output(self):
        print (chr(int(self.tape[self.pointer])), end='')
    def inp(self, inp):
        self.tape[self.pointer] = inp
    def check_loop(self):
        if self.tape[self.pointer] == 0:
            return 0
        else:
            return 1
{% endhighlight %}

#### Part 2: Clojure
Writing an interpreter for brainfuck in a language like python was fairly straight forward. The conceptual "tape" made sense as a class, and it was easy to keep track of all that state. Despite this, I thought it would be a good excersize to try and write one in a Lisp (Clojure). 

The first difficulty here taking a very stateful and mutating program I had created, and writing it in a language that doesn't allow mutation.

Keeping with a functional paradigm, using recursion should match well for this type of interperter. Clojure has the function `loop` which makes this easy. Below is a (terrible) way to right a function to produce the nth fibonacci number using the `loop` function in Clojure. (just don't pass it 0,1,2) 
{% highlight clojure %}
(defn fib [n]
  (loop [n1 1 n2 0 iter 2]
    (if (= iter n)
      n1
      (recur (+ n1 n2) n1 (inc iter)))))
{% endhighlight %}

Simply, `loop` just acts as a target point for the `recur` function. I believe it helps to unwrap the recursion into more general iteration -- avoiding stack overflows (citation needed*). 

Having this in mind, the following function will evaluate the subset of a brainfuck program without `[, ], ,`
{% highlight clojure %}
(defn eval [text]
  (loop [cells [0] cell-pointer 0 inp 0]
    (condp = (get text inp)
      \> (recur (conj cells 0) (inc cell-pointer) (inc inp))
      \< (recur cells (dec cell-pointer) (inc inp))
      \+ (recur (update-in cells [cell-pointer] inc) cell-pointer (inc inp))
      \- (recur (update-in cells [cell-pointer] dec) cell-pointer (inc inp))
      \. (do
           (print (char (nth cells cell-pointer)))
           (recur cells cell-pointer (inc inp)))
      nil cells
      (recur cells cell-pointer (inc inp)))))
{% endhighlight %}
Here, the memory, memory pointer, and instruction pointer are all initialized in the `loop` bindings. `condp` acts as a sort of switch statement, or a more concise `if/else` chain. 

It was an interesting experience writing these two different programs. They both do the same thing, but in vastly different ways. In Python, it felt most natural to handle all the state in a class, and mutate over it as new instructions came in. While in Clojure, the whole interpreter was able to be represented in a recursive way. 

[brainfuck-wiki]: https://en.wikipedia.org/wiki/Brainfuck

