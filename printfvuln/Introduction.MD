# Printf Vulnerabilities

a printf vulnerability is caused because printf mixes data and "code".

Typically a printf vulnerability is present when you have something like 

```
buf[30];
read(0, buf, 30); 
printf(buf);
```
Sending a variable into printf modified by a user is dangerous as it could leak the stack or arbitrarily write at some area in memory.

Before playing with printf vulnerabilities you should first know what format specifiers do. 
(I will not enumarate all of them there is a lot I will list out the common ones and the ones relevant to printf vulnerability challenges)

## %s Arb read
%s looks into an address and dereferences the address.

```
(pretend this is gdb)

rsp 0xff787804 0x1
    0xff787808 0xff787814 -> 0x74
    0xff78780c 0x1
    0xff787810 0x29
    0xff787814 0x74
    0xff787818 0x89
```
if my %s was looking at address 0xff787808 we would print out 0x74. As that address contains an address that points to another address and so %s dereferences it.
Now if we were to use %s on 0xff787814 a segfault would occur as it would attempt to derefence address 0x74 which is not a valid address.

**TLDR %s is doing print(\*addr) instead of print(addr) in C notation.**

## %p Arb read

%p is a bit like %s except it does not dereference.
```
rsp 0xff787804 0x1
    0xff787808 0xff787814 -> 0x74
    0xff78780c 0x1
    0xff787810 0x29
    0xff787814 0x74
    0xff787818 0x89
```
Coming back to my previous example if my %p was looking at 0xff787808 it would simply print out 0xff787814.
If it was looking at 0xff787818 it would print out 0x89.

**TLDR %p reads straight off the stack with no dereferencing. Think of p like pointer because it prints out a pointer (that is what is named after).**

## %x Arb Read

%x is like %p except it prints it out as if it was a hex value rather than a pointer like %p 

its output and behavior is smilar to the output %p minus having 0x prepended

**TLDR %x literally %p except no 0x at the front (josh may have more insight)**

## %n Arb Write

%n the arbitrary write, it functions in a similar way to %s in the way that it derefences an address and then writes the amount of characters printed to said address. 
I do not know the purpose of this monster. (it was meant for counting printed characters who asked for this?)

```
rsp 0xff787804 0x1
    0xff787808 0xff787814 -> 0x74
    0xff78780c 0x1
    0xff787810 0x29
    0xff787814 0x74
    0xff787818 0x89
```
For references lets say my format string was 
"123456789.%n" (lets also drop the $ notation for this example I will bring up the $ later)

Back to gdb, lets say my %n was looking at 0xff787808, lets also say that in the same printf I also printed out 123456789. %n would write 10 into 0xff787814 transforming the stack to

rsp 0xff787804 0x1
    0xff787808 0xff787814 -> 0xa
    0xff78780c 0x1
    0xff787810 0x29
    0xff787814 0xa
    0xff787818 0x89

we succesfully wrote 10 to address 0xff787814 with that format string. Now if we were looking at any other address that did not an address to dereference it would segfault, much like %s.

Now with all the important specifiers out of the way we can talk about $ notation.

## $ notation

$ notation is what you will use to what data your format specifiers access.

This will access a stack address relative to to the printf 

Note printf starts from the beginning the parameters of printf and then goes downwards from there.
for example if you had printf("%4$d", var1,var2,var3,var4); it would print out var4 so you must find where the first parameter to printf would be and that would be the base from where you start accessing stack addresses.

```
#disclaimer the parameters for printf are not always top of stack, you would figure it by checking out what address the vararg corresponds to. Then start index after the address 
rsp 0xff787804 0x1                   #printf %1$p
    0xff787808 0xff787814 -> 0xa     #printf %2$p
    0xff78780c 0x1                   #printf %3$p
    0xff787810 0x29                  #printf %4$p
    0xff787814 0xa                   #printf %5$p
    0xff787818 0x89                  #printf %6$p
```
Ripping this from gdb

https://github.com/Joel-Flores123/joel-flores123.github.io/blob/main/printfvuln/images/printf.png

Notice the address on vararg? 

https://github.com/Joel-Flores123/joel-flores123.github.io/blob/main/printfvuln/images/printf2.png

Now looking back at the stack that address appears in 2 places. This is just a coincidence, but starting the address with the checkmark you can access data. The checkmarked address is index 1 and you can
index downwards from there.

This is only an introduction ill probably post some examples later on.





