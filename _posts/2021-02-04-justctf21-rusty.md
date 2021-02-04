---
layout: post
title: justCTF 2020 - rusty
tags: writeup re justctf20
date: 2021-02-04 09:52 +0100
---
#### Note

The challenge was solved after the CTF ended, but before any writeups had been published.

## Challenge

> Looking at Rust code in disassembler/decompiler hurts, so... look somewhere else.
> 
> [Link to rusty.exe](https://ams3.digitaloceanspaces.com/justctf/4c3f61cd-6ab1-47a1-9fa6-be2fd34c8d08/rusty.exe)

## Solution

As the name of the challenge suggests, the executable is a Rust-based binary, which targets Windows / 64-bit.
The challenge also hints that it might be infeasible to just reveng the normal .text sections and find the flag.

Running `rusty.exe`, yields the ubiquitous "_Give me the flag:_" prompt. Entering a random value, it returns "_lol. That's not even close._"

### Analyzing rusty.exe with Ghidra

Opened `rusty.exe` in Ghidra and used the _Search_ -> _For Strings..._ dialog to look for "flag".
Found the string and used the _Show references to Address_ context menu option to navigate to the sole
usage (`LEA RAX, [PTR_s_Give_me_the_flag:_140020638]`).

It successfully decompiled a function in the Decompiler window, but something was amiss; my window turned
bright (I use a dark Ghidra theme) which indicated that Ghidra hadn't created a proper function for it.

In the Listings view, I used the _Create Function (F)_ option on the first instruction (`PUSH RBP`), which
fixed that.

Scrolling a bit down in the function, I discovered the following section:

```c
    // ...
    if (*(char *)ppuStack200 == 'j') {
      if (*(char *)((longlong)ppuStack200 + 1) == 'c') {
        if (*(char *)((longlong)ppuStack200 + 2) == 't') {
          if (*(char *)((longlong)ppuStack200 + 3) == 'f') {
            if (*(char *)((longlong)ppuStack200 + 4) == '{') {
              if (*(char *)((longlong)ppuStack200 + 0x36) == '}') {
    // ...
```

We can see that the flag is supposed to start with `jctf{` and end with `}` which is a bit odd,
since the flags in the JustCTF should have started with `justCTF{`.

It also reveals the flag size (`0x36 + 1`).

---

Alas, this is how far I got before the competition ended, but after I got a hint that the flag was hidden elsewhere in the `exe`, I had to return to crack this open.

---

### Meet the DOS stub

The Windows Portable Executable format (PE) is designed to be backwards compatible - that is, if a user
tries to execute a Windows binary under DOS, it will normally just tell the user:

> This program cannot be run in DOS Mode.

This message is not output from the OS; it is generated from a small 16-bit DOS executable section in the
start of the .exe that is known as the _DOS stub_.

_Normally_ this stub contains the message above and basically `MOV AL, 9 / INT 0x21` which is the DOS equivalent syscall for writing to a File Descriptor, in this case STDOUT.

But when I inspected the header fields in the program, I noticed that the header (Ghidra marks this section as `e_program`) was 4368 bytes long!

#### Extracting e_program

The easiest way (using partially Ghidra) to extract the e_program header section into a separate binary,
was to select the whole region, and use _Copy Special_ -> _Byte String_ on the context menu, paste it
into a temporary file, or directly into something like CyberChef, choose the _From Hex_ recipe, and
download the decoded bin. Of course it is possible to cut out the relevant bytes using command line tools,
but this time I wanted to check if there was a feasible way of doing it with Ghidra. It would be nice if
Ghidra had a way to export a section directly as raw, verbatim bytes, or even better, re-import it into
the project as a separate file.

#### Analyzing the DOS stub

Since I cut out the header region without any PE/MZ header, some non-conventional steps are needed when
importing it into Ghidra.

![Import raw binary 1](/assets/images/posts/justctf21_rusty/rusty-import-dos-stub.bin-1.png "Import raw binary in Ghidra #1")

![Import raw binary 2](/assets/images/posts/justctf21_rusty/rusty-import-dos-stub.bin-2.png "Import raw binary in Ghidra #2")

To find out where the first instruction is, we need to go back and inspect the MZ header.

`e_ip` (Initial IP value) is set to `0x8a` - but what does it offset from?

Two DOS terms we need to know about:

* A _paragraph_ is `16` bytes (`0x10`)
* A _page_ is `512` bytes (`0x200`)

Turns out that we need to take page alignment into account. Since the header up to the `e_program`
section is `0x40` bytes, the current page will be padded with `0x200 - 0x40 = 0x1c0 (448)` bytes
before the page(s) for the working set of the executable.

![Ghidra byte listing](/assets/images/posts/justctf21_rusty/rusty-dos-stub-bin-listing-1.png "Ghidra raw listing of rusty DOS stub")

At `0x1c0`, we discover the bytes `This program cannot be run in DOS Mode.$`. Notice the
`$` at the end? Strings handled by the syscalls of `INT 0x21` are terminated by a dollar - not by `0`
as you might be used to - or Pascal-style, with the string length stated as a prefix byte.

At `0x1c0 + 0x8 = 0x24a`, we find `0xfc` which translates to the `CLD` (Clear Direction Flag) in 8086 assembly.

![Ghidra assembly listing](/assets/images/posts/justctf21_rusty/rusty-dos-stub-bin-listing-2.png "Ghidra assembly listing of rusty DOS stub")

Since demo scene-related coding was something I did during the mid-90'es, I quickly realize that I'm
familiar with most of the stuff going on.

Before analyzing the binary further, I installed `dosbox-debug` and fired it up.

I mounted a local directory containing `rusty.exe`:

```shell
Z:\>MOUNT r /home/larsw/justctf/re/rusty
Z:\>r:
R:\>RUSTY.EXE
```

![Flames effect](/assets/images/posts/justctf21_rusty/rusty-flames.png "Flames effect!")

Yay! Instant déjà vu!

Before exiting the program (with either ESC or Enter), I noticed that for each key I pressed,
a asterisk was displayed and merged into the flame effect.

Back to Ghidra. As I mentioned earlier, most of the decompiled assembly made sense to me, so
I started to clean up/rename functions until I felt that I had identified all the key components.
Apart from setting up "mode 13h" (320x200/8-bit) and a nice palette (doing I/O on port 0x3c7/0x3c8),
there was also the standard `wait_for_vsync` routine, and of course the logic for generating some
pseudo random bytes and creating three copies of them on the bottom of the screen, before doing passes
to smooth out the rows above. All this was done in a never ending loop with a check for keyboard status
/ read of ASCII code from keyboard (`INT 0x16`).

Looking a bit harder on the keyboard loop, I realized that _ESC_ would quit the application, pressing _Enter_
would quit - but do something more first, backspace would move a pointer backwards in an input buffer, and pressing any other key would append them to the same input buffer (including moving the pointer). At most `39` (`0x27`) bytes could be entered into the buffer - entering more would simply be ignored.
The input buffer was located at `ds:0x66`.

#### Figuring out the secret key sequence

I had hoped that the initial seed for the pseudo random generator routine would be the key sequence
that had to be input in order to get the exe to output the flag upon exit, but that wasn't the case.

As mentioned in the previous section, pressing Enter would exit the program, but not straight away.
There was a bit of code that seemed to work on both the input buffer and another buffer - one located
at `ds:0x34`. Looking at the bytes in that area, gave a hint that there was something hidden there -
quite possibly the flag. The buffer was also `0x27` long, and was terminated with an additional `$`.

I fired up `dosbox-debug` and started `rusty.exe` in the debugger with `DEBUG RUSTY.EXE`, set up a couple of breakpoints (Ex. `BP cs:0132`) and continued the execution flow (_F5_).

Having the cleaned up pseudo code in Ghidra side-by-side was a great help understanding what was going
on.

There was an additional piece of code that would calculate a checksum by adding up
all the bytes in the `0x34`-buffer, and check to see if it equalled `0xd9f`. If it was
correct, the program would output the buffer, else it would output the `This program cannot` message.

Rewritten in python, the pseudo code looked like this:

```python
flag = [] # 0x34
input = [] # 0x60
flag_length = 0x27

for i,v enumerate(input):
    for j in range(i,flag_length):
        flag[j] ^= input[i]

if sum(flag) == 0xd9f:
  print(flag.decode())
else
  print("This program cannot be run in DOS mode.")
```

E.g. the first byte of the obfuscated flag was XOR'ed with only the first byte of the input, while
the second one was XOR'ed with both the first and second - and so on.

With no other a priori knowledge, this exercise would have been nearly impossible to solve, but since
we knew that all flags started with `justCTF{` I set up the following in a spreadsheet:

![Deciphering](/assets/images/posts/justctf21_rusty/rusty-deciphering.png "Deciphering rusty flag in a spreadsheet")

Guessing the additional `gram` gave `justCTF{just`, so I was certain that was the correct continuation.

Two seconds later - and a big facepalm, I realized that the secret input had to be:

> This program cannot be run in DOS Mode.

*Doh!*

Always remember _Occam's razor_ (loosely paraphrased):

> The simplest explanation is usually the right one.

## Flag

`justCTF{just_a_rusty_old_DOS_stub_task}`

## Video

<iframe width="420" height="315" src="http://www.youtube.com/embed/615Xcp_2cgE" frameborder="0" allowfullscreen></iframe>

## Resources

* http://www.keithholman.net/pe-format.html
* https://wiki.osdev.org/PE#DOS_Stub
* https://www.hanshq.net/fire.html
* https://en.wikipedia.org/wiki/Occam%27s_razor