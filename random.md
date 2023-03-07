# random
This is the 6th challenge in the [ Toddler's Bottle ] section of pwnable.kr.

Immediately, we are greeted with :
```
Daddy, teach me how to use random value in programming!

ssh random@pwnable.kr -p2222 (pw:guest)
```

After SSHing into the remote machine, we find three files with `ls`:
```cmd
random@pwnable:~$ ls
flag  random  random.c
```
With `file`, we observe that **random** is a 64-bit executable.
```cmd
random: setuid ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=f4eac0a1434a84aef72dfabfc1f889e6f6f73023, not stripped
```
Let's take a look at **random.c**:
```c
#include <stdio.h>

int main(){
        unsigned int random;
        random = rand();        // random value!

        unsigned int key=0;
        scanf("%d", &key);

        if( (key ^ random) == 0xdeadbeef ){
                printf("Good!\n");
                system("/bin/cat flag");
                return 0;
        }

        printf("Wrong, maybe you should try 2^32 cases.\n");
        return 0;
}
```
**Interpreting the code**

We observe that `main` is accepting a decimal integer `key` to be compared with a `random` integer using the bitwise XOR operator `^` in C. 
In order to get the flag, we must figure out a `key` value that will make `key ^ random` equal to `0xdeadbeef`.

While it may seem impossible to get a `key` with the value of `random` being different every run, it turns out that the `rand()` glibc function is [not entirely random](https://0xstubs.org/libcs-random-number-generators-and-what-to-be-aware-of-when-seeding-them/).

In fact, because the function is not provided a seed, it will result in the same "random" integer every time we run the program.

To find this value, we can create a simple test program on the pwnable.kr machine.
First,
```cmd
mkdir /tmp/randtest
cd /tmp/randtest
nano test.c
```
Then, we can create our test random integer generator with no seed:
```c
#include <stdio.h>

int main(){
        unsigned int random;
        random = rand();

        printf("%x\n", random);
        return 0;
}
```
Now, compile and run it!
```cmd
random@pwnable:/tmp/randtest$ gcc -o test test.c
random@pwnable:/tmp/randtest$ ./test
6b8b4567
random@pwnable:/tmp/randtest$ ./test
6b8b4567
random@pwnable:/tmp/randtest$ ./test
6b8b4567
```

You can see, no matter how many times we run the file, `random` is `6b8b4567`.

By performing an XOR on the two values, we can find our `key` value!
(You can use an online calculator or do it by hand.)
Now, we know:
```c
random = 0x6b8b4567 = 0110 1011 1000 1011 0100 0101 0110 0111
         0xdeadbeef = 1101 1110 1010 1101 1011 1110 1110 1111
key    = 0xb526fb88 = 1011 0101 0010 0110 1111 1011 1000 1000
```
However, the input to random is an integer, so we need a decimal value:
```c
key = 3039230856
```

Now, we can send our payload!
```cmd
random@pwnable:~$ ./random
3039230856
Good!
Mommy, I thought libc random is unpredictable...
```
![Yay!](https://media.giphy.com/media/U5IXAxSQYXWfEu9ZDY/giphy.gif)
