---
layout: post
title: "Fixing a Security Bug by Changing a Function Signature"
author: Molly Howell
date: 2021-09-29
categories: 
  - "bug-bounty"
  - "firefox-internals"
---

 

### **Or: The C Language Itself is a Security Risk, Exhibit #958,738**

This post is aimed at people who are developers but who do not know C or low-level details about things like sign extension. In other words, if you're a seasoned pro and you eat memory safety vulnerabilities for lunch, then this will all be familiar territory for you; our goal here is to dive deep into how integer overflows can happen in real code, and to break the topic down in detail for people who aren't as familiar with this aspect of security.

# **The Bug**

In July of 2020, I was sent [Mozilla bug 1653371](https://bugzilla.mozilla.org/show_bug.cgi?id=1653371) (later assigned [CVE-2020-15667](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-15667)). The reporter had found a segfault due to heap overflow in the library that parses [MAR files](https://wiki.mozilla.org/Software_Update:MAR)[1](#footnote-1), which is the custom package format that’s used in the Firefox/Thunderbird application update system. So that doesn’t sound great. (spoiler: it isn't as bad as it sounds because that overflow happens after the MAR file has had its signature validated already)

# **The Fix**

[The patch](https://hg.mozilla.org/mozilla-central/rev/b79b6cc78248) I wrote for this bug consists entirely of changing [one function signature](https://searchfox.org/mozilla-central/rev/dafb74eec8028248324018e8cd32b93808e3fd5c/modules/libmar/src/mar_read.c#29) in a C source file from this:

```
static int mar_insert_item(MarFile* mar, const char* name, int namelen,
                           uint32_t offset, uint32_t length, uint32_t flags)
```

to this:

```
static int mar_insert_item(MarFile* mar, const char* name, uint32_t namelen,
                           uint32_t offset, uint32_t length, uint32_t flags)
```

I swear that is the entire patch. All I had to do was change the type of one of this function’s parameters from `int` to `uint32_t`. Can that change really fix a security bug? It can, and it did, and I’ll explain how. We have some background to cover first, though.

# **Background**

The problem here comes down to numbers and how computers work with them, so let’s talk a bit about that first[2](#footnote-2). Since the bug is in a file written in the C language, our discussion will be from that perspective, but I am going to try to explain things so that you don’t need to know C or much at all about low-level programming in order to understand what happened.

## **Binary Numbers**

Any number that your computer is going to work with has to be stored in terms of binary bits. The way those work isn’t as complicated as it might seem.

Think about how you write a number in decimal digits using place value. If we want to write the number one thousand, three hundred, and twelve, we need four digits: 1,312. What does each one of those digits mean? Well the rightmost 2 means… 2. But the 1 next to that doesn’t mean 1, it means 10. You take the digit itself and multiply that by 10 to get the value that’s being represented there. And then as you go through the rest of the digits, you go up by another power of 10 for each one. The 3 doesn’t mean either 3 or 30, it means 300, because it’s being multiplied by 100. And the leftmost 1 gets multiplied by 1000.

Guess what? Binary numbers work the same way. The only difference is, since binary only has two different digits, 0 and 1, it doesn’t make any sense to use powers of 10; there’d be loads of numbers we couldn’t write, anything greater than 1 but less than 10 couldn’t be represented. So instead of that, we use powers of 2. Each successive digit isn’t multiplied by 1, 10, 100, 1000, etc., it’s multiplied by 1, 2, 4, 8, etc.

Let’s look at a couple of examples. Here’s the number twelve in binary: 1100. Why? Well, let’s do the same thing we did with our decimal example, multiply each digit. I’ll write out the whole thing this time:

```
1100
│││└─ 0 x (2 ^ 0) = 0 x 1 = 0
││└── 0 x (2 ^ 1) = 0 x 2 = 0
│└─── 1 x (2 ^ 2) = 1 x 4 = 4
└──── 1 x (2 ^ 3) = 1 x 8 = 8

0 + 0 + 4 + 8 = 12
```

There we go! We got 12. For each digit, we multiply its value by the power of 2 for that place value location (and the multiplication is pretty darn easy, because the only digits are 0 and 1), and then add up all those results. That’s it!

### **Binary Addition**

Now, what if we need to do some math? That’s pretty much all computers are any good at, after all. Let’s say we want to add something to a binary number.

Well, we know how to do that in decimal: you add up each digit starting from the lowest one and carry over into the next digit if necessary. If you read the last section, you can probably guess what I’m about to say: that’s exactly what you do in binary too. Except again it’s even easier because there’s only two different digits.

Let’s have another simple example, 13 + 12. First we have to write both of those numbers in binary; we already know 12 is 1100, so 13 should just be one more than that, 1101. We’ll add them up the same way we add decimal numbers by hand:

```
  1100
+ 1101
------
 ?????

The first two digits are easy, 0 + 1 = 1, and 0 + 0 = 0.

  1100
+ 1101
------
 ???01
```

But now we have 1 + 1. Where do we go with that? There’s no 2. Well, just like in decimal, we have to carry out of that digit; the sum of 1 and 1 in binary is 10 (because that’s just binary for 2), so that means we need to write a 0 in that column and carry the 1.

```
  1
  1100
+ 1101
------
 ??001
```

Only one digit to go. Again, it’s 1 + 1, but now we have a 1 carried over from the previous digit. So really we have to do 1 + 1 + 1, which is 3 but in binary that’s 11. This is the last column now, so we don’t have to worry about carries anymore, we can just write that down:

```
  1
  1100
+ 1101
------
 11001
```

And we’re done! 1100 + 1101 = 11001. And to prove we got the right answer, let’s convert 11001 back to decimal, the same way we did before:

```
11001
││││└ 1 x (2 ^ 0) = 1 x  1 =  1
│││└─ 0 x (2 ^ 1) = 0 x  2 =  0
││└── 0 x (2 ^ 2) = 0 x  4 =  0
│└─── 1 x (2 ^ 3) = 1 x  8 =  8
└──── 1 x (2 ^ 4) = 1 x 16 = 16

1 + 0 + 0 + 8 + 16 = 25
```

So now we _know_ we were right; 12 + 13 = 25, and 1100 + 1101 = 11001. That’s how you add numbers in binary.

### **Signed Integers and Two’s Complement**

So far we’ve only talked about positive numbers, but that’s not all computers can handle; sometimes you also need negative numbers. But you don’t want _every_ number to potentially be negative; a lot of the kinds of things that you need to keep track of in a program just cannot possibly be negative, and sometimes (as we’ll see) allowing certain things to be negative can be actively harmful.

So, computers (and many languages, including C) provide two different kinds of integers that the programmer can select between whenever they need an integer: “signed” or “unsigned”. “Signed” means that the number can be either negative or positive (or zero), and “unsigned” means it can only be positive (or zero)[3](#footnote-3).

What we’ve been talking about up to now are unsigned integers, so how do signed integers work? To start with, the first bit of the number isn’t part of the number itself anymore, it’s now the “sign bit”. If the sign bit is 0, the number is nonnegative (either zero or positive), and if the sign bit is 1, the number is negative. But, when the sign bit is 1, we need a couple extra steps to convert between binary and decimal. Here’s the procedure.

1. Discard the sign bit before doing anything else.
2. Invert all the other bits in the number, meaning make every 1 a 0 and vice versa.
3. Convert that binary number (the one with the bits flipped) to decimal the usual way.
4. Add 1 to that result.

This operation, with the inversion and the adding 1, is called [“two’s complement”](https://en.wikipedia.org/wiki/Two%27s_complement), and it’ll get you the value of the negative number. Let’s go through another simple example.

Let’s say we have a signed 8-bit integer and the value is 11010110. What is that in decimal? Well, we see right away that the sign bit is set, so we need to take the two’s complement. First, we need to flip all the bits except the sign bit, so that gets us 0101001. Now we convert that to decimal and add 1.

```
0101001
││││││└ 1 x (2 ^ 0) = 1 x  1 =  1
│││││└─ 0 x (2 ^ 1) = 0 x  2 =  0
││││└── 0 x (2 ^ 2) = 0 x  4 =  0
│││└─── 1 x (2 ^ 3) = 1 x  8 =  8
││└──── 0 x (2 ^ 4) = 0 x 16 =  0
│└───── 1 x (2 ^ 4) = 1 x 32 = 32
└──────

1 + 0 + 0 + 8 + 0 + 32 = 41

41 + 1 = 42
```

Now just remember to add back the negative sign, and we get -42. That’s our number! 11010110 interpreted as a signed integer is -42.

#### _Why?_

Why do we bother with any of this? Why not do something simple like have the sign bit and then just the regular number[4](#footnote-4)? Well, the two’s complement representation has one huge advantage: you can completely disregard it while doing basic arithmetic. The exact same hardware and logic can do arithmetic on both unsigned numbers and signed two’s complement numbers[5](#footnote-5). That means the hardware is simpler, which means it’s smaller, cheaper, and faster. That mattered more in the early days of digital computers, which is why two’s complement caught on as the standard, and it’s still with us today.

#### _Sign Extension_

There’s one other neat trick two’s complement let’s us do that we need to talk about. Integers in computers have a fixed “width”, or number of bits that are used to represent them. Wider integers can represent larger (or more negative) numbers, but take up more space in the computer’s memory. So to balance those concerns, languages like C give the programmer access to a few different bit widths to choose from for their integers.

So, what happens if we need to do some arithmetic between integers that are different widths, or just pass an integer into a function that’s narrower than the function expects? We need a way to make an integer wider. If it’s unsigned, that’s easy; copy over the same value into the lower (right-hand) bits and then fill in the new high bits with 0’s, and you’ll have the same value, just now with more bits.

But what if we need to widen a signed integer? Two’s complement’s here to save the day with a solution called [“sign extension”](https://en.wikipedia.org/wiki/Sign_extension). It turns out all we have to do to make a two’s complement integer wider is copy over the same value into the low bits and then fill in the new high bits with copies of the sign bit. That’s it.

It’s easy to see why that’s correct if we think about how two’s complement works. If the number is positive (the sign bit is 0), then it’s the same as for an unsigned number, we’ll fill in the new space with all zeroes and nothing changes. And if the number is negative (the sign bit is 1), then we’ll fill in the new space with 1 bits, but the two’s complement operation means those bits all get inverted into 0’s when we need to get the number’s value, so _still_ nothing changes. These simple, efficient operations are why two’s complement is so neat, despite seeming weird and overcomplicated at first.

### **Hexadecimal Numbers**

I’m going to use a few hexadecimal numbers in this article, but don’t worry, I’m not going to try to teach you how to work in a whole different number system yet again. You can think of hexadecimal as a shorthand for binary numbers. Hexadecimal (“hex” for short) uses the decimal digits 0-9 and also the letters A-F, for 16 possible digits total. Since each digit can have 16 values, each one can stand in for four binary digits.

Also, hex numbers in C and elsewhere are written starting with 0x. That’s not part of the number, it’s just telling you that the thing after it is written in hex so that you know how to read it.

You don’t need to know how to do any arithmetic directly on hex numbers or anything like that, just see how they convert to binary bits. Here’s the conversions of individual hex digits to binary bits:

```
Binary  Hex
======  ===
 0000    0
 0001    1
 0010    2
 0011    3
 0100    4
 0101    5
 0110    6
 0111    7
 1000    8
 1001    9
 1010    A
 1011    B
 1100    C
 1101    D
 1110    E
 1111    F
```

## **Implicit Conversions in C**

In C, unlike some languages, there are a bunch of different types that represent different ways of storing numbers; basically, every kind and size of number that CPU’s can work with has its own type in C. There’s also a “default” integer type, which is called int. How many bits are in an int depends on the C compiler you’re using (and on its settings)[6](#footnote-6), but it is guaranteed by the language standard to be signed.

Since C has so many different kinds of numbers, it’s common to need to convert between them. It’s so common in fact that the language designers decided to make those conversions mostly automatic. That means that, for instance, this code compiles and runs as you’d probably expect:

```
#include <math.h> // to get the declaration for sqrt()

long long geometric_mean(int a, int b) {
  return sqrt(a * b);
}

int main() {
  int a = 42;
  long b = 13;
  double mean = geometric_mean(a, b);
  return mean;
}
```

Even though none of the types in that code match up at all, the compiler just makes everything work for us. Nice of it, eh? These automatic “fixes” are called implicit conversions, and [the rules for how they work](https://en.cppreference.com/w/c/language/conversion) are long and not always very intuitive. This is a pretty major gotcha of C programming, because it happens without the programmer even seeing it, you just have to know these things are happening and realize all the implications.

# **How the Bug Works**

That should be all the background we need to understand what went wrong here. Now, let’s have another look back at that original, unpatched function declaration:

```
static int mar_insert_item(MarFile* mar, const char* name, int namelen,
                          uint32_t offset, uint32_t length, uint32_t flags)
```

The first two parameters are an internal data structure and a text string, they aren’t relevant here. But after that we see an int parameter, which is meant to contain the length of the string parameter (in C, strings don’t know their own length, the programmer has to keep track of that if they need it).

A few lines into the mar\_insert\_item function, we find this call:

```
memcpy(item->name, name, namelen + 1);
```

I’ll explain what this line is for before we move on. The mar\_insert\_item function is part of a procedure that reads the index of all the files contained in the MAR package (it’s kind of like a ZIP file, it can contain a bunch of different files and compress them all, and you can extract the whole thing or just individual files). mar\_insert\_item is called repeatedly, once for each compressed file, and each call adds one entry to the index that’s being gradually built up. This specific line just copies the file’s name into that index entry; memcpy of course is short for “memory copy”, and its parameters are the destination to copy to (which is the name field of the item we’re adding to our index), the source to copy from (the name string was passed into mar\_insert\_item in the first place), and the amount of memory that needs to be copied, in bytes. That last parameter is where everything goes wrong.

What do you think would happen if mar\_insert\_item is called with namelen set to the highest positive value it can store, which is 0x7fffffff? Well then, in this one line of code, the program does all of these things:

1. A 1 gets added to `namelen`[7](#footnote-7). But I just said `namelen` already has the highest positive value it can store, so something has to give. The C language standard doesn’t define what happens in this case, but in practice what you get on most computers is… the addition just happens anyway. So we get the value 0x80000000. But `namelen` is a signed integer, and that value has its sign bit set! We’ve added 1 to a positive number and it transformed into a negative number. -2,147,483,648 to be precise[8](#footnote-8). Computers are weird. And we’re not even done yet.
2. `memcpy` takes a 64-bit value, so our temporary value has to get extended from 32 bits to 64. That means a sign extension; we take the most significant bit, which is a 1, and copy it into 32 new bits, getting us the value 0xFFFFFFFF80000000. Remember, sign extension preserves the two’s complement value, so the decimal version of that number is still -2,147,483,648, it didn’t change during this step.
3. The length parameter that `memcpy` takes is also supposed to be unsigned, so now that the value has been extended to 64 bits, we take those bits and interpret them as an unsigned number. We no longer have -2,147,483,648, we now have positive 9,223,372,036,854,775,807. As a byte length, that’s over a trillion terabytes[9](#footnote-9). Fair to say that’s more bytes than we could have really meant to be copying here.
4. Finally, memcpy is called, and it starts trying to copy from name into `item->name`. But because of that sign extension and unsigned reinterpretation, we can see that it’s going to try to copy waaaaay more bytes than are actually there. So what `memcpy` ends up doing is copying all the bytes that are there (`memcpy` does its best for us even when we feed it junk), and then… crashing the program.

And that’s the bug; the updater crashes right here.

## **How the Fix Works**

Now, with all that background, the fix makes perfect sense. Changing the parameter’s type means that the conversion to unsigned happens at the time `mar_insert_item` is called, and at that point the value being passed in is still a positive number, so converting it then is harmless (in fact it’s just nothing, that operation doesn’t do anything at all at that point). And then the + 1 is done to an unsigned number, so it’s harmless too, and there’s no sign extension to ever do because the thing being passed to `memcpy` is no longer signed. Everything gets a lot simpler to understand, and simultaneously more correct.

# **Takeaways**

## **Don’t Use C**

Implicit conversions are a misfeature. What they give you in convenience is more than erased by the potential for invisible bugs. More recently designed languages tend to be more strict about this sort of thing, Rust for instance just [doesn’t have these kinds of implicit conversions at all](https://doc.rust-lang.org/rust-by-example/types/cast.html), but C is from the 1970’s and It Made Sense At The Time™. But in C these things can’t really be avoided, they’re baked into the language. I’d very much recommend using another language for any new programs you work on, for this and a variety of other reasons[10](#footnote-10).

## **Layers of Security**

This bug wasn’t exploitable in practice, partly because it’s just in an awkward place to exploit, but also because Firefox requires update files to be digitally signed by Mozilla or they won’t be read (beyond the minimum needed to check the signature), much less applied. That means that anybody wanting to attack Firefox users via this bug would also have to compromise Mozilla’s build infrastructure and use it to sign their own malicious MAR file. Having that additional layer of security makes most issues surrounding MAR files much much less concerning.

## **You Can Do Systems Programming**

Something I’ve hoped to get across (and I acknowledge this may not be the ideal topic to make this point, but it’s an important point to me) is that low-level (“systems”) programming isn’t magic or really special in any way. It’s true there’s a lot going on and there’s lots of little details, but that’s true for any kind of programming, or anything else involving computers at all to be honest. Everything involved here was invented and built by people, and it can all be broken down and understood. And that’s the message I want to sign off with: you can do systems programming. It’s not too hard. It’s not too complicated. It’s not limited to just “experts”. You are smart and capable and you can do the thing.

* * *

1\. A fair question to ask here would be why we even have our own package format. There’s a few reasons and you can read the original discussion from back when the format was first introduced if you’re interested, but the main benefit nowadays is that we’re able to locate and validate the package’s signature before really having to parse anything else. In fact, the bug that this post is about doesn’t get hit until after the MAR file has passed signature validation, so it could only be exploited using either a properly signed MAR or a custom build of Firefox/Thunderbird/whatever other application that disables MAR signing. [↩︎](#footnote-1-back)

2\. I’m only going to talk about integers, because numbers that have a decimal or fraction part work very differently (and can be implemented a few different ways), and they aren’t relevant here. [↩︎](#footnote-2-back)

3\. You almost never need numbers that can only be either negative or zero, so neither hardware nor languages generally support those, you’d just have to use a signed integer in that case. [↩︎](#footnote-3-back)

4\. That is a real thing called [signed magnitude](https://en.wikipedia.org/wiki/Signed_number_representations#Signed_magnitude_representation) and it is used for certain things, but not standard integers in modern computers. [↩︎](#footnote-4-back)

5\. If you’re curious about the math that explains why this is the case, I’ll direct you to [Wikipedia’s proof](https://en.wikipedia.org/wiki/Two's_complement#Why_it_works); I’ve spent enough time in the weeds for one blog post already. [↩︎](#footnote-5-back)

6\. Theoretically int is meant to be whatever size the computer hardware you’re compiling your program for finds most convenient to work with (its “word size”), so it would be 32 bits on a 32-bit CPU and 64 bits on a 64-bit CPU. In practice though, for backwards compatibility reasons, int is usually 32 bits on all but pretty specialized hardware. It’s best never to depend on int being any particular size and to use the type that specifically represents a particular size if you need to be sure; for instance if you know you need exactly 32 bits, use int32\_t, not int. [↩︎](#footnote-6-back)

7\. If you don’t know C, you might be wondering what the + 1 is even for. It’s a little out of scope for this post, but in short, since as we mentioned earlier C strings don’t keep track of their length, if you don’t store that length off somewhere (and typically you don’t), you need some other way to find where the string ends. That’s done by adding one character made up of all zero bits to the end of the string, called a “null terminator”, so when you’re reading a string and you encounter a null character, then you know the string is over. Most C coding conventions have you leave the terminator out of the length, so whenever you’re doing something that needs to account for the terminator (like copying it, because then you have to copy the terminator also), you have to add 1 to the length so that you have space for it. C programming is full of fiddly details like this. [↩︎](#footnote-7-back)

8\. This problem shows up so often and is such a common source of security bugs that it gets its own name, [integer overflow](https://en.wikipedia.org/wiki/Integer_overflow). On Wikipedia you'll find lots of famous examples and different ways to combat the issue. [↩︎](#footnote-8-back)

9\. AKA one yottabyte. I swear that is really what it’s called. [↩︎](#footnote-9-back)

10\. Yes, I acknowledge there are certain circumstances where you really must write things in C, or maybe C++ if you’re lucky. If you have one of those situations, then you already know whatever I could tell you. If you don’t, then don’t use C. And don’t @ me. [↩︎](#footnote-10-back)
