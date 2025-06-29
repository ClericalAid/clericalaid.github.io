---
layout: default
title: "Debugging deep dive (vocode)"
date: 2024-03-07
---

# Debugging deep dive (vocode)
Date: 2024-03-07

### TL;DR
1. Get a backtrace
2. Ignore standard library files in the backtrace
3. Set breakpoints near locations identified in the backtrace
4. Step through code and explore variables

# The error
When trying to get vocode to run, I would get an error from the underlying C code. I would find out later that this error was being encountered in the `whisper.cpp` library. I have never worked with python or C in any professional capacity, but it was a good night to get my hands dirty. So away I went!

The given error resembled the following:
```
terminate called after throwing an instance of 'std::bad_alloc'
  what():  std::bad_alloc
Aborted
```

# Getting more information
## Which library is even failing?
That error message was not very helpful. So we have to dig deeper.

We know it's a problem in the C library because `std::bad_alloc` is not a python error, as it's trying to allocate memory.

In gdb, we can use the following to get information on C libraries while debugging a python script.
```
gdb --args python my_srcript.py
```

Then we need to type `run`, and when we hit our error, we can get a backtrace with `bt`. The backtrace was:

```
#0  __pthread_kill_implementation (threadid=<optimized out>, signo=signo@entry=6, no_tid=no_tid@entry=0) at ./nptl/pthread_kill.c:44
#1  0x00007ffff7aa81cf in __pthread_kill_internal (signo=6, threadid=<optimized out>) at ./nptl/pthread_kill.c:78
#2  0x00007ffff7a5a472 in __GI_raise (sig=sig@entry=6) at ../sysdeps/posix/raise.c:26
#3  0x00007ffff7a444b2 in __GI_abort () at ./stdlib/abort.c:79
#4  0x00007fffd5aa0a2d in ?? () from /lib/x86_64-linux-gnu/libstdc++.so.6
#5  0x00007fffd5ab1f5a in ?? () from /lib/x86_64-linux-gnu/libstdc++.so.6
#6  0x00007fffd5aa05d9 in std::terminate() () from /lib/x86_64-linux-gnu/libstdc++.so.6
#7  0x00007fffd5ab21d8 in __cxa_throw () from /lib/x86_64-linux-gnu/libstdc++.so.6
#8  0x00007fffd5aa0649 in ?? () from /lib/x86_64-linux-gnu/libstdc++.so.6
#9  0x00007fff8f0f1e7b in whisper_full_with_state () from /home/chrish/workspace/external/whisper.cpp/libwhisper.so
...
```

## First look at the backtrace
We now have the backtrace for our error. If you are new to seeing these, they will look very daunting. I would recommend trying to ignore the standard library stuff (libstd).

We see some references to `whisper` in that backtrace. To setup vocode locally, they require you to build `libwhisper.so`, which is a part of the `whisper.cpp` project, locally. So this is a good clue on where to go next.

## Compile whisper with debug information
In order to get extra debug information from `whisper.cpp`, we need to compile it with a flag in order to get debug symbols. I added a `-g` to the `CXXFLAGS` variable in the makefile and then reran `make libwhisper.so`.
```
CXXFLAGS = -I. -I./examples -O3 -DNDEBUG -std=c++11 -fPIC -g
```

## Run the script in gdb again
With our library recompiled, we rerun the script to get a new backtrace with debug symbols. Said backtrace can be found below. The most important parts are lines #15, #16, and #21. This is because they point directly to the `whisper.cpp` library that I compiled on my machine - which we noticed in the first backtrace.

```
#0  __pthread_kill_implementation (threadid=<optimized out>, signo=signo@entry=6, no_tid=no_tid@entry=0) at ./nptl/pthread_kill.c:44
#1  0x00007ffff7aa81cf in __pthread_kill_internal (signo=6, threadid=<optimized out>) at ./nptl/pthread_kill.c:78
#2  0x00007ffff7a5a472 in __GI_raise (sig=sig@entry=6) at ../sysdeps/posix/raise.c:26
...
Cutting out a few frames
...
#9  0x00007fff8eef1e7b in std::__new_allocator<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> > >::allocate (this=0x7ffc89037b20, __n=140737326330672)
    at /usr/include/c++/13/bits/new_allocator.h:151
#10 std::allocator_traits<std::allocator<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> > > >::allocate (__n=140737326330672, __a=...) at /usr/include/c++/13/bits/alloc_traits.h:482
#11 std::_Vector_base<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> >, std::allocator<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> > > >::_M_allocate (__n=140737326330672, this=0x7ffc89037b20) at /usr/include/c++/13/bits/stl_vector.h:378
#12 std::_Vector_base<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> >, std::allocator<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> > > >::_M_create_storage (__n=140737326330672, this=0x7ffc89037b20) at /usr/include/c++/13/bits/stl_vector.h:395
#13 std::_Vector_base<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> >, std::allocator<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> > > >::_Vector_base (__a=..., __n=140737326330672, this=0x7ffc89037b20) at /usr/include/c++/13/bits/stl_vector.h:332
#14 std::vector<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> >, std::allocator<std::vector<whisper_grammar_element, std::allocator<whisper_grammar_element> > > >::vector (
    __a=..., __n=140737326330672, this=0x7ffc89037b20) at /usr/include/c++/13/bits/stl_vector.h:554
#15 whisper_grammar_init (i_start_rule=1, n_rules=140737326330672, rules=0x7ffff4b9170d) at whisper.cpp:4173
#16 whisper_full_with_state (ctx=<optimized out>, state=<optimized out>, params=<error reading variable: Cannot access memory at address 0x8>, samples=<optimized out>, n_samples=<optimized out>)
    at whisper.cpp:5214
...
#21 _ctypes_callproc (pProc=pProc@entry=0x7fff8eef6980 <whisper_full(whisper_context*, whisper_full_params, float const*, int)>, argtuple=argtuple@entry=0x7ffc8b5036a0, flags=4353, argtypes=argtypes@entry=0x0,
```

## Understanding the backtrace
As mentioned above, we see calls to the whisper library on lines #15, #16, and #21.

Don't worry if you don't see it, but the backtrace shows that the C code is attempting to allocate a very large vector. And this is happening because the variable `n_rules` is a very large number.

You're not expected to immediately see the problem from the backtrace. And that's okay, because even if we lack this intuition, we can still get to the answer slowly.

Thanks to the debug flags, we see the following: `whisper.cpp:5214`. This means that one of the last lines of code ran before encountering this error was line 5214 in the file `whisper.cpp`.

## Setting a breakpoint
In `gdb` we can set a breakpoint to see what's going on in this area. I like to set my breakpoints one line before the error - so 5213 instead of 5214:
```
break path/to/whisper.cpp:5213
```

```C
// Line 5213 begins here
                if (params.grammar_rules != nullptr) {
                    decoder.grammar = whisper_grammar_init(params.grammar_rules, params.n_grammar_rules, params.i_start_rule);
                } else {
                    decoder.grammar = {};
                }
```

Then run the program, again, hit the breakpoint, and start looking at the variables to see what's going on.

```
(gdb) frame
#0  whisper_full_with_state (ctx=<optimized out>, state=<optimized out>, params=...,
    samples=<optimized out>, n_samples=<optimized out>) at whisper.cpp:5213
5213                    if (params.grammar_rules != nullptr) {
(gdb) p params.grammar_rules
$1 = (const whisper_grammar_element **) 0x7ffc5c018d30
(gdb) p params
$2 = {strategy = (unknown: 0x3354), n_threads = 0, n_max_text_ctx = 812755209, offset_ms = 0,
  duration_ms = 1485378656, translate = 85, no_context = 85, no_timestamps = false,
  single_segment = false, print_special = 49, print_progress = 14, print_realtime = 14,
  print_timestamps = 143, token_timestamps = 255, thold_pt = 1.52774392e+17,
  thold_ptsum = 4.59121429e-41, max_len = 1544007928, split_on_word = 184, max_tokens = 1481426912,
  speed_up = 85, debug_mode = 85, audio_ctx = 1456761840, tdrz_enable = 85,
  initial_prompt = 0x6 <error: Cannot access memory at address 0x6>, prompt_tokens = 0x0,
  prompt_n_tokens = 255172004,
  language = 0xbb800000004 <error: Cannot access memory at address 0xbb800000004>,
  detect_language = 184, suppress_blank = 11, suppress_non_speech_tokens = false,
  temperature = -nan(0x7fffff), max_initial_ts = 0, length_penalty = 0,
  temperature_inc = 9.05043025e+14, entropy_thold = 3.0611365e-41, logprob_thold = 4.20389539e-42,
  no_speech_thold = 0, greedy = {best_of = 1479722720}, beam_search = {beam_size = 21845,
    patience = 2.04386567e+17}, new_segment_callback = 0xbb8, new_segment_callback_user_data = 0xc1e,
  progress_callback = 0x3078, progress_callback_user_data = 0x2ee0,
  encoder_begin_callback = 0xbb800000c1e, encoder_begin_callback_user_data = 0xbb700000050,
  abort_callback = 0x7ffc890378f0, abort_callback_user_data = 0x55555832c6e0,
  logits_filter_callback = 0x7ffc5c07b0f8, logits_filter_callback_user_data = 0x7ffc89037a80,
  grammar_rules = 0x7ffc5c018d30, n_grammar_rules = 140722607198048, i_start_rule = 140722607198000,
  grammar_penalty = -7.05596484e-30}
```

This `params` variable actually gets passed in from python. And looking in the C code, the `n_grammar_rules` variable is indeed very large. Let's see what value this is supposed to be back in python.

## Getting the breakpoint in vocode
To get a breakpoint here, I looked at the method that was being run in `whisper.cpp` when the error happened. Looking at the stack trace, we see a call to `whisper_full` at #21.

This is the first reference to whisper that happens in the stacktrace. So in the `vocode` repository, I searched for all calls to `whisper_full`.

There was one reference. So we set a breakpoint with `pdb.set_trace()`, rebuild the project locally and install it. Then, running this project again, we will hit our breakpoint.

```
(Pdb) p params.n_grammar_rules
*** AttributeError: 'WhisperFullParams' object has no attribute 'n_grammar_rules'
(Pdb)
```

## Making sense of everything
So afterwards, I had to check the `WhisperFullParams` object and it seemed like it's supposed to match the shape of the C struct in `whisper.cpp`. This is because this parameter is being passed in from python into the C code via the `ctypes` library, and C is very picky with how structs are shaped.
