# Float On
## Challenge Description:
Category: Misc.

> I cast my int into a double the other day, well nothing crashed, sometimes life's okay.  
> We'll all float on, anyway: [float_on.c](https://github.com/enh-code/CTF-writeups/edit/main/angstromCTF/2021/float_on.c).  
> 
> Author: solomonu (branding by kmh)

## How to Solve it:
After you've recovered from the high of finding out someone else loves ["Float On" by Modest Mouse](https://www.youtube.com/watch?v=CTAud5O7Qqk) and recognizing it, we can take a look at the contents of the file 
in question.
```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <assert.h>

#define DO_STAGE(num, cond) do {\
    printf("Stage " #num ": ");\
    scanf("%lu", &converter.uint);\
    x = converter.dbl;\
    if(cond) {\
        puts("Stage " #num " passed!");\
    } else {\
        puts("Stage " #num " failed!");\
        return num;\
    }\
} while(0);

void print_flag() {
    FILE* flagfile = fopen("flag.txt", "r");
    if (flagfile == NULL) {
        puts("Couldn't find a flag file.");
        return;
    }
    char flag[128];
    fgets(flag, 128, flagfile);
    flag[strcspn(flag, "\n")] = '\x00';
    puts(flag);
}

union cast {
    uint64_t uint;
    double dbl;
};

int main(void) {
    union cast converter;
    double x;

    DO_STAGE(1, x == -x);
    DO_STAGE(2, x != x);
    DO_STAGE(3, x + 1 == x && x * 2 == x);
    DO_STAGE(4, x + 1 == x && x * 2 != x);
    DO_STAGE(5, (1 + x) - 1 != 1 + (x - 1));

    print_flag();

    return 0;
}
```

The first thing that jumped out to me was the union since I have never really seen it much in practice.  
In case you don't know what a union is, a union is like a struct, however all the variables occupy the same space in memory.  
The different types are really just different ways of interpreting the data inside the union.  
  
Specificially, the union allows a conversion from an unsigned 64 bit integer into a 64 bit floating point (`double`);  
To see this in action, and generally a great tool for this challenge and studying IEEE 754 (the floating-point standard), check out this [link](https://www.h-schmidt.net/FloatConverter/IEEE754.html) (this is a 32 bit float, be warned).
  
Looking at the next point of interest, the `DO_STAGE` macro, the second and third lines of the function take in an unsigned integer:
```c
scanf("%lu", &converter.uint);
```
And then convert it to a `double` and store it into `x` (defined as `double` in `main`):
```c
x = converter.dbl;
```
It finally performs the check specified when it was called.  
It's important to note that the check happens as a `double`.  

### Stage 1 (Normalcy)
Addressing the first condition: `x == -x`, the only number that can fit this (with normal math anyway) is 0.  
`Stage 1: 0`  
Well, that was easy, let's try another!  

### Stage 2 (NaN Antics)
The second test: `x != x`. Uh oh. If this was normal math, the universe breaks right about here.  
But this isn't normal math. ***THIS IS FLOATING POINT MATH.***  
Floating point math has a lot of quirks (and is confusing in general, good luck). The two interesting cases are `NaN` (not a number) and `inf` (infinity).  
  
`NaN` represents all floating-point values that cannot exist (including complex numbers, just think the real number system).  
`NaN` has a few properties:
  1. `NaN != NaN` is `true`.
  2. `0 / 0 == NaN`.
  3. `inf * 0 == NaN`.
  4. `inf / inf == NaN`.
  5. Basically anything weird with infinity.  

The first point is obviously the answer to the question, but how do you represent NaN as an unsigned 64 bit integer?

Well, NaN is typically represented in binary as all 1s or 0 followed by all 1s. We could type the maximum number in decimal for a 64 bit integer, but I'm feeling lazy, so let's be sneaky and type -1 instead.  
Why -1? Well, it's kind of off topic, but if you run -1 through a signed int to binary converter, it changes the number to all 1s--exactly what we're looking for.  
`Stage 2: -1`  
What's next? The one that made me hate this challenge.  
  
### Stage 3 (Infinity in Finite Space)
The third test: `x + 1 == x && x * 2 == x`. Just that first expression made me want to give up. (To be honest, I didn't know about `NaN` or `inf` until I debugged the heck out of this.)  
Obviously, this is another universe breaking test, so normal numbers are out the window. We'll have to default on our two friends.  
Here's another [link](https://www.cs.uaf.edu/2012/fall/cs301/lecture/10_24_weirdfloat.html) to a table that defines what various operations can be done to get what outcomes (bottom of page).  
Here's the thing, though, we already established that `NaN != NaN`. And the table confirms the notion that `NaN + 1` is the same as `NaN` (but not equivalent to because `NaN` is stupid).  
Then who else do we turn to? Why, infinity, of course!  
  
Infinity kind of just ignores the numbers you do operations on (except 0 in some cases, see above).  
But the big difference is that `inf == inf` which is exactly what we're looking for. Thanks to our earlier table, we can safely assume that using infinity will work.  

Infinity dives into the the way floating-points are actually structured. I won't go too in depth (I barely have it down myself), but we'll cover the basic structure.
```
63     62         52   51                                                  0
 X      XXXXXXXXXXX     XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
 ^           ^                                   ^
 |           |                                   |
Sign      Exponent                            Mantissa
```
Each `X` is a bit. The sign denotes whether it is positive or negative, the exponent is where the whole number is typically stored (`2^exp - 1023`, where `exp` is the decimal representation of the exponent), and the mantissa is the part that holds the fractional value.  
That is a bit to take in, but for our purposes, infinity is represented as all of the exponent bits set to 1. Now that becomes `0111111111110000000000000000000000000000000000000000000000000000`.  
Much too large to comprehend, so plugging it into a binary to unsigned int converter gives us `9218868437227405312`.  
`Stage 3: 9218868437227405312`  
  
### Stage 4 ("What's Just Before Infinity?")
I'm going to be honest, I guessed on this one, and I still don't really know why it works. Here's my thinking, though.  
The fourth test: `x + 1 == x && x * 2 != x`. Looked similar to the last one, so it can't be too different. The first expression ruled out `NaN`.
  
Fiddling with the exponent, I thought, "what if I just changed the lowest exponent bit and tried that?" To my shock, it worked!  
In binary, this would be `0111111111100000000000000000000000000000000000000000000000000000`. In decimal, it's `9214364837600034816`.  
`Stage 4: 9214364837600034816`  
  
### Stage 5 (The Final Trial)
The final test: `(1 + x) - 1 != 1 + (x - 1)`.

Had this been an equal sign, any real number would have solved this. However, it's Angstrom, so we can't have anything super easy. But we do have one tool that always comes in handy when logic dies in front of our face: `NaN`.  
We've already established that -1 is a great way to achieve `NaN`, so in the grinder it goes!
`Stage 5: -1`

## Flag
`actf{well_we'll_float_on,_big_points_are_on_the_way}`  
  
## Final Thoughts
 - SERIOUSLY LISTEN TO THAT SONG IT'S ONE OF THE BEST PICK ME UPS EVER
 - 80s people must have been really desperate if they're going to cram all that data into such a little space (and we didn't even touch denormalized floats).
 - Floating-point numbers are a lot more complex, so much so that they're part of an internet standard and not the language itself (that's why bare bones systems don't have floats, one must create the logic to handle them, IEEE 754 is just a good way to do it).
 - This challenge really forces the contestant to truly understand how the bits affect floating point numbers, a great miscellaneous exercise.

