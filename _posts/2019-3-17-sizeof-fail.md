---
layout: post
title: My allcation fail
---

I’ve stumbled upon a nasty bug recently.
The bug was completely my fault and I was surprised that the compiler didn’t protected me by issuing an error or warning.
I bug was nasty because it seemed like a libc function was corrupting a structure I allocated ealier.
Our codebase is fairly large and it took some tries to reproduce it with a smaller example, but I managed to make it happen, so here’s the code:
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define STR "Have a really nice day!"
#define B 80
#define N 5

struct str {
        short fields[B];
        char s[0];
};

int main(void)
{
        struct str *s;
        char *sd[N];
        int i;

        printf("sizeof(str)=%ld\n", sizeof(struct str));

        s = calloc(1, sizeof(s) + sizeof(STR) + 1);
        if (!s)
                return 1;

        strcpy(s->s, STR);

        printf("s=%s\n", s->s);

        for (i = 0; i < N; ++i) {
                sd[i] = strndup(s->s, strlen(STR) + 1);
                printf("s=%s sd[%d]=%s\n", s->s, i, sd[i]);
        }

        for (i = 0; i < N; ++i)
                free(sd[i]);

        free(s);

        return 0;
}
```

And the output of this code is:
```
sizeof(str)=160
s=Have a really nice day!
s=Have a really nice day! sd[0]=Have a really nice day!
s=Have a really nice day! sd[1]=Have a really nice day!
s=Have a really nice day! sd[2]=Have a really nice day!
s=ce day! sd[3]=Have a
s=ce day! sd[4]=ce day!
```

Please keep in mind that this is an example and the names of the structure and variables should reflect their usage in real life.

So looking at the code, you can see that I allocate a structure, write and string to the last field and then use the strdup libc function to duplicate this string a few times, printing the original string and the duplicate each time.
And after the 3rd duplication, it seems like both the original string and the duplicate string gets corrupted.
But how? The only function I call is ‘strdup’, is it possible that there’s a bug in it?

Looking at the code, you can see I’m using quite a few “c” tricks – using “calloc” instead of “malloc” because it clears the structure and I can skip the “memset(str, 0, ...)” step.
I also use sizeof(<name of variable>) instead of the name of the type (“struct str”) to short the line of code.

But this is where my mistake lied – The structure “str” used to be allocated on the stack as so: “struct str s = {};”, then I decided to allocate it on the heap with ‘calloc’ and what I forgot to do was to change the “sizeof(s)” to “sizeof(*s)”.

Also note that there are other not recommended ‘c’ things I’m doing in this example – the usage of zero length arrays (or flexible array member in C99) can be confusing and error prone. And the use of ‘strcpy’ is not recommended as it has not bound checks and can lead to corruption and security issues.

