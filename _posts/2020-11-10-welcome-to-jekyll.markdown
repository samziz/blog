---
layout: post
title: "What's the metal?"
date: 2020-10-11
categories: Foundations
---

When we say "close to the metal?", what's the metal: what happens when your code runs? How does the computer run your code, how is memory allocated and garbage-collected, how do concurrency and parallelism work, and what other questions should we know the answers to?

I wrote this for a friend, to illustrate roughly what's involved in writing the kind of web- or app-based application code that most software engineers are likely to write today. This should cover lots of the topics that would be relevant to other popular areas, like embedded or security engineering, but the areas it emphasises are more tailed toward web- or app-based product engineering. It doesn't specifically address mobile engineering, but since both iOS and Android are more or less analogous to MacOS and Linux respectively - on a low level - most of the same information should be applicable to those platforms.

> This was written for a friend with a Mac, so it makes a lot of mention of Mac-related GUI tools to avoid dipping too much into the command line for a first-time explorer. Feel free to ignore these parts, or treat it as an exercise for the reader to go and find the Linux or Windows equivalents.

## How writing code works

This section is going to cover:

[Saying hello to the terminal](#saying-hello-to-the-terminal)\
[Concurrency and parallelism](#concurrency-and-parallelism)\
[Garbage collection](#garbage-collection)

The point of writing code is to take some input, do some computations, and return an output. It might also be to store 'state' – the programming term for 'memory', in the sense that Facebook's servers have 'state' consisting of all your posts, and a machine learning program has 'state' in the form of the weights of each neuron in the neural net.

Practically all computers run an operating system. It's perfectly possible to run code without an operating system (for one thing, that's what an operating system does!), but an operating system contains all the code to handle an unimaginably vast range of different processor types, displays, network cards, and much more. The operating systems you use are all overwhelmingly likely to be 'Unixes', meaning they descend from the Unix operating system. MacOS is officially a Unix, whereas Linux is Unix-y - that's actually the technical term.

When you run your code, you tell the operating system to start a 'process' to run your program[^caveat]. On a modern computer, you will probably have a CPU with something like 4 cores. Tons of processes will be running all the time on those cores. Even if you don't have anything 'open' on your display, the operating system will be constantly running processes doing all manner of things, like Bluetooth, rendering graphics to display on-screen, polling the Apple servers for any notifications, and so forth. If you open Activity Monitor on your Mac you can see how many are running.

#### Saying hello to the terminal

For bonus points, open your terminal (search 'Terminal', and add it to your dock, because you'll need it in future), enter `ps -A`, and hit the return key to run the command. This will print several hundred processes. This is a good time to explain piping. In your 'shell' - which is the program underpinning the Terminal app - you can use piping to take the output of one command, such as the `ps -A` command we just ran, and supply it as the input to another command. Think of it a bit like that Human Centipede film. In order to know _exactly_ how many processes are running, use the 'pipe' character (`|`) to pipe the output of the `ps` program into another program called `wc` (**w**ord **c**ount), which is used for counting stuff like words, characters, and lines: `ps -A | wc -l`. This should give you a single number telling you how many lines `ps -A` printed - in other words, how many processes are running right now.

#### Concurrency and parallelism

I have 427 processes running now, on my MacBook which has eight CPU cores. How does that work? My computer can't be _literally_ running 427 different programs simultaneously, because we would need 427 processors (i.e. CPU cores[^core]) for that. What happens instead is that the operating system has something called a [scheduler](<https://en.wikipedia.org/wiki/Scheduling_(computing)#Short-term_scheduling>) which rapidly switches between different programs. If you imagine each program as a chess board, each processor as a player on the chess board (ignore the opponent for the purpose of the metaphor), and each instruction as a 'turn', then the scheduler's job is to move the players - meaning the cores - between chess boards. My CPU is 2.3Ghz, meaning there are 2.3bn CPU cycles (on each cycle, each core can execute one instruction), totalling 18.4bn cycles across all the cores.

Where possible, the CPU will try to pre-empt services if they're currently 'idle' waiting for some I/O (input/output) to complete. Using the chess board analogy, the scheduler wants to avoid wasting cores on a chess board where it's the opponent's turn and the opponent hasn't moved yet. If one process sends a request over the internet, it tells the 'kernel' (the core of the operating system, which comes with a lot of 'ready-made' code to do things like that) to send the request. The process says something along the lines of "call me back when the WiFi card receives a packet in response, and then I'll process the response". The program is now blocked for [about 150ms](https://gist.github.com/jboner/2841832), or in other words 345 million CPU cycles. There's no point telling the CPU to run that process until the networking code in the kernel tells the scheduler that it's received its response and is ready to do more work. So instead, the scheduler will try to find the highest-priority process among all those that are waiting (which may be e.g. another process that was blocked on I/O but _has_ now received a response).

What I described above is referred to as _concurrency_. Concurrency is the property of rapidly flicking between different tasks. _Parallelism_ is the property of doing several tasks simultaneously. The subtle distinction between these two terms is somewhat confusing because in the normal English language the two words are synonymous, and 'concurrency' doesn't imply rapidly alternating between several things - but you'll just have to remember that it does in this context. And as far as our 427 tasks are concerned, the scheduling process involves some parallelism (in that most modern CPUs have more than 1 core, so computers do run several processes in parallel across multiple cores), but it relies _much_ much more on concurrency.

#### Garbage collection

When you write code, you often assign values to variables. It will often look something like this:

```python
def binary_search(array, target, _offset=0):
  length = len(array)

  middle = int(length / 2)
  middle_value = array[middle]

  if middle_value > target:
    return binary_search(array[0:middle], target, middle+_offset)
  elif middle_value < target:
    return binary_search(array[middle:length], target, middle+_offset)
  else:
    return middle+_offset

index = binary_search([-20, 10, 21, 99, 121, 150, 200, 942, 1023], 150)
```

This is a binary search algorithm. It's a simple trick: we have a list of numbers that's already sorted (i.e. we know it's in order, from smallest to largest) and we want to find the index (i.e. the place in the array, first, last, or some place in between) of a particular number. So we take the middle number: if it matches we return it, if it's too large then we repeat with the first half of the array, and if it's too small we repeat with the second half. We don't need to go into the exact performance of binary search here, since we're not discussing algorithms yet.

What we're interested in here is garbage collection. In more or less every programming language, you can assign values to variables. This means that we can do what we did in this function, and create a variable like `middle` and give it a value like `int(length / 2)`. I'm going to tell a bit of a simplified story here, pretending that the only place a variable can be stored is in your RAM stick. This is not quite true - there are memory areas called caches in your CPU, used because they are extremely fast to access - but those details don't make any difference to what we're discussing here.

OK, caveats out of the way. When you assign a variable like `middle`, Python reserves an area in your RAM for that variable. Let's say you have an 8GB stick of RAM, which means you have 8589934592 bytes. You can write or read data to RAM by supplying the offset to start from (say, the 20th 8-bit-long byte along, therefore 160) as well as the data to write.

Python's interpreter, which executes your program, is written in C. In C, you use a memory allocator (you can even write one!) which requests a big chunk of memory from the operating system, and then every time a bit of code wants to use some memory, it tells the allocator how much memory it wants to use, and the allocator finds a nice spot in the chunk (in a clever way, so as to avoid having any tiny unusable fragments in between two blocks of allocated memory).

Each time you call `binary_search`, it will need to create another `middle` variable, each time with a slightly different value. And each time, Python's interpreter will ask its allocator to give it an offset in the chunk that it can write to, and it will write to it, and the allocator will mark that chunk as 'in use'. Now, eventually we'll hit a problem where the allocator doesn't have any more chunks to give. It can request even more memory from the OS for it to parcel out, but eventually even _that_ will run out. What the interpreter needs to do (in C, where you must do that manually) is tell the allocator that it's not using that memory in any more.

Now, in C, as you can infer, when you're not using a variable any more, you have to _tell_ the computer that - you need to call the `free` function to 'free up' that memory. This is extremely tedious, and it causes lots of bugs. If you forget to free a variable, you'll have a memory 'leak': this means you keep allocating more memory but never freeing it (every time the function is called), and you eventually run out of memory and crash. If you do the opposite, and accidentally free your memory too early and then try to use it after telling the allocator it's free, it's possible some other code will have overwritten that chunk of memory, meaning it's now different and you'll have a very very hard-to-spot bug.

All of the above means that Python - wisely - tries to avoid what's called _manual memory management_, where you have to tell the computer when you're done using a variable. There are some programming languages that are written in a clever way, where when you run your code, it can essentially figure out where it needs to insert some invisible code that frees the memory. Rust and Swift both implement this, in slightly different ways. Python uses a different technique called 'garbage collection'. Garbage collection means that there's some other invisible code in your program, which runs at the same time as your code, and keeps track of every variable.

Garbage collection revolves around the idea of whether a variable is 'reachable'. For instance, every time we execute the `binary_search` function, we know that when that function returns (i.e. finishes), the variable `middle` is no longer reachable. This is because of the rules of Python: a variable you declare (create) while inside a function is only 'in scope' - in other words it can only be accessed - from other code inside that same function call.

Python has an excellent library called `gc` that lets you interact with its garbage collector _from inside your function while it's running_. Let's try printing out all the values of all the variables from _every time_ this function has been executed, at least the ones that Python's garbage collector hasn't wiped up yet:

```python
import gc

def binary_search(array, target, _offset=0):
  length = len(array)

  middle = int(length / 2)
  middle_value = array[middle]

  print('* Current values for the variables in this function *')
  print([o for o in gc.get_objects() if type(o).__name__ == 'list' and middle_value in o])

  if middle_value > target:
    return binary_search(array[0:middle], target, middle+_offset)
  elif middle_value < target:
    return binary_search(array[middle:length], target, middle+_offset)
  else:
    return middle+_offset

index = binary_search([-20, 10, 21, 99, 121, 150, 200, 942, 1023], 150)
```

If you do this, you might see something like this:

```
* Current values for the variables in this function *
[['gc', 'binary_search', 'array', 'target', 0, '_offset', 'length', 'len', 'array', 'middle', 'int', 'length', 2, 'middle_value', 'array', 'middle', 'print', '* Current values for the variables in this function *', 'print', 'o', 'o', 'gc', 'get_objects', 'list', 'type', 'o', '__name__', 'o', 'middle_value', 'print', '\n\n\n', 'target', 'middle_value', 'binary_search', 'array', 'middle', 'length', 'target', 'middle', '_offset', 'middle', '_offset', 'target', 'middle_value', 'binary_search', 'array', 0, 'middle', 'target', 'middle', '_offset', 'index', 'binary_search', 20, 10, 21, 99, 121, 150, 200, 942, 1023, 150, -20], [-20, 10, 21, 99, 121, 150, 200, 942, 1023]]


* Current values for the variables in this function *
[['gc', 'binary_search', 'array', 'target', 0, '_offset', 'length', 'len', 'array', 'middle', 'int', 'length', 2, 'middle_value', 'array', 'middle', 'print', '* Current values for the variables in this function *', 'print', 'o', 'o', 'gc', 'get_objects', 'list', 'type', 'o', '__name__', 'o', 'middle_value', 'print', '\n\n\n', 'target', 'middle_value', 'binary_search', 'array', 'middle', 'length', 'target', 'middle', '_offset', 'middle', '_offset', 'target', 'middle_value', 'binary_search', 'array', 0, 'middle', 'target', 'middle', '_offset', 'index', 'binary_search', 20, 10, 21, 99, 121, 150, 200, 942, 1023, 150, -20], [-20, 10, 21, 99, 121, 150, 200, 942, 1023], [121, 150, 200, 942, 1023]]


* Current values for the variables in this function *
[['gc', 'binary_search', 'array', 'target', 0, '_offset', 'length', 'len', 'array', 'middle', 'int', 'length', 2, 'middle_value', 'array', 'middle', 'print', '* Current values for the variables in this function *', 'print', 'o', 'o', 'gc', 'get_objects', 'list', 'type', 'o', '__name__', 'o', 'middle_value', 'print', '\n\n\n', 'target', 'middle_value', 'binary_search', 'array', 'middle', 'length', 'target', 'middle', '_offset', 'middle', '_offset', 'target', 'middle_value', 'binary_search', 'array', 0, 'middle', 'target', 'middle', '_offset', 'index', 'binary_search', 20, 10, 21, 99, 121, 150, 200, 942, 1023, 150, -20], [-20, 10, 21, 99, 121, 150, 200, 942, 1023], [121, 150, 200, 942, 1023], [121, 150]]
```

If you look at the final few arrays, you can see that each time the function executes, there's another slice of the array. This is because Python's garbage collector hasn't cleaned up these variables yet. They are still in scope, because we keep calling this function from inside itself: a technique called [recursion](https://en.wikipedia.org/wiki/Recursion). Python's garbage collector keeps track of the references to each variable. Once we exit the function, the garbage collector - the next time it runs - will be able to look at the currently executing code and see that the function calls which created those variables have since returned, and therefore those variables are out of scope and gone forever. It will then tell the memory allocator to mark the associated memory as free again, so it can be used by other code.

This should explain a bit about how memory allocation works in code, as well as the different ways of managing memory. To recap, the most basic, close-to-the-metal way of allocating code is what C does: the programmer must say when a variable is created and when it's no longer usable and its memory should be freed. This is fast and means you don't have to do garbage collection at the same time as running your actual code, but it's bug-prone. Then you have languages like Rust and Swift, where code written in that language has to obey certain rules, and as a result it's possible for the compiler to know where to automatically insert code to free up memory. And finally we have garbage collected languages, like Python, which don't require you to free memory manually, or to use a language where you need to structure your code in such a way that the language can free memory automatically, _but_ this comes at the expense of running garbage collection while your code is running - competing with your code for CPU cycles.

---

[^caveat]: Caveat: I'm going to talk about this as if we're using a compiled language rather than an interpreted one, just to remove one additional layer of complexity (i.e. that with interpreters, the CPU is running the _interpreter_ program, which uses your code as instructions to tell it what to do as it runs).
[^core]: Each core in a [multi-core](https://en.wikipedia.org/wiki/Multi-core_processor) processor architecture can _essentially_ be called a processor. Some people may quibble with that claim, since cores all live on the same board and can share things like memory caches, which fully separate processors would not. But in practice, separate cores can basically be described as separate processors.
[^warn]: Note that the thing it tells you to do is generally not a good idea - it's telling you to use the command `curl` to fetch the text of a web page, and pipe that straight into `bash`, your shell, which directly executes the text of the web page as code. This is very common but very dangerous, because often websites will encourage you to do this with a website which may eventually be abandoned by its owner, at which point the site can (and often will) be acquired by savvy criminals who alter the page so it contains malicious code. You should generally read through code and understand it before executing it. In this case, though, it's OK: Homebrew is very very widely used software, and the code it refers to is hosted on GitHub, which is not going to be abandoned any time soon.
