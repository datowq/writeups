# collision
This is the second challenge in the [ Toddler's Bottle ] section of pwnable.kr.

Immediately, we are greeted with :
```
Daddy told me about cool MD5 hash collision today.
I wanna do something like that too!

ssh col@pwnable.kr -p2222 (pw:guest)
```

After SSHing into the remote machine, we find three files with `ls`:
```cmd
col@pwnable:~$ ls
col  col.c  flag
```
With `file`, we observe that **col** is a 32-bit executable.
```cmd
col@pwnable:~$ file col
col: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=05a10e253161f02d8e6553d95018bc82c7b531fe, not stripped
```
Let's take a look at **col.c**:
```c
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
    int* ip = (int*)p;
    int i;
    int res=0;
    for(i=0; i<5; i++){
        res += ip[i];
    }
    return res;
}

int main(int argc, char* argv[]){
    if(argc<2){
        printf("usage : %s [passcode]\n", argv[0]);
        return 0;
    }
    if(strlen(argv[1]) != 20){
        printf("passcode length should be 20 bytes\n");
        return 0;
    }

    if(hashcode == check_password( argv[1] )){
        system("/bin/cat flag");
        return 0;
    }else
        printf("wrong passcode.\n");
    return 0;
}
```
**Interpreting the code**

We observe that `main` is accepting a 20-byte-long password that is sent through the function `check_password` and checked to see if it matches with the hashcode `0x21DD09EC`.

`check_password` is casting the 20-byte-long password from a `const char*` to an `int*`.

The `for` loop then adds five divisions of the 20-byte-long password and outputs their sum.

If we recall, **col** is a 32-bit executable, so integers will take up 4 bytes, which matches the 5 divisions that the `for` loop sums.

**Now, all we have to do is prepare our payload:**

First, let's convert the hex hashcode `0x21DD09EC` to decimal and divide it into 5 parts with:
```python
col@pwnable:~$ python3
Python 3.5.2 (default, Jan 26 2021, 13:30:48)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> int(0x21DD09EC)
568134124
>>> 568134124/5
113626824.8
>>> 568134124-(113626824*4)
113626828
```
The first four parts are equal divisions of `113626824` with the fifth part as the remainder `113626828`. 

To turn these decimal values into a 20-byte-long password, we can pack them into little-endian byte code using the `struct` python library.
```python
>>> import struct as s
>>> s.pack('<i', 113626824)
b'\xc8\xce\xc5\x06'
>>> s.pack('<i', 113626828)
b'\xcc\xce\xc5\x06'
```
Now, we can send our payload!
```cmd
col@pwnable:~$ ./col $(python -c "print(b'\xc8\xce\xc5\x06'*4+'\xcc\xce\xc5\x06')")
daddy! I just managed to create a hash collision :)
col@pwnable:~$
```
![Yay!](https://media.giphy.com/media/U5IXAxSQYXWfEu9ZDY/giphy.gif)
