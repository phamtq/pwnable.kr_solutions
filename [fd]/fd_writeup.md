# [Toddler's Bottle] : [fd]

## Challenge
![A cute Anime style dhjkshfdspink slime](images/fd.png)  

A wild **[fd]** has appeared and like any JRPG, this is for  beginners. It issues me the following challenge:

![First challenge: Mommy! what is a file descriptor in Linux? try to play the wargame your self but if you are ABSOLUTE beginner, follow this tutorial link: https://youtu.be/971eZhMHQQw. ssh fd@pwnable.kr -p2222 (pw:guest)](images/fd_challenge.png)  

## Status

![Super Mario RPG, defeating a chomp](images/slayed.gif)

Slayed!!! +1 XP

## Approach
Okay, first challenge! From the description I figure it'll be their lowest level of difficulty. What that meant for someone self described as an "okay coder, but bad programmer" I don't know. I have some experience writing C & C++ programs from tutorials (mostly) but that's probably going to be minimally helpful. 

So as a first step let's see what's in the directory:  

```bash
fd@ubuntu:~$ ls -alh
total 48K
drwxr-x---   5 root fd     4.0K Apr  1 14:50 .
drwxr-xr-x 118 root root   4.0K Jun  1 12:05 ..
d---------   2 root root   4.0K Jun 12  2014 .bash_history
-r-xr-sr-x   1 root fd_pwn  15K Mar 26 13:17 fd
-rw-r--r--   1 root root    452 Mar 26 13:17 fd.c
-r--r-----   1 root fd_pwn   50 Apr  1 06:06 flag
----------   1 root root    128 Oct 26  2016 .gdb_history
dr-xr-xr-x   2 root root   4.0K Dec 19  2016 .irssi
drwxr-xr-x   2 root root   4.0K Oct 23  2016 .pwntools-cache
```

Let's take a look at what's in the `flag` file. It couldn't be that easy, could it?

```bash
fd@ubuntu:~$ cat ./flag
cat: ./flag: Permission denied
```

Yep, definitely not that easy. Taking a look at the file permission for the file it shows that it's readable by the owner and group but not other. Additionally, it looks like it's owned by `root` and the group `fd_pwn`.

The program `fd` must stand for Linux file descriptors as hinted in the challenge description. I know that there's:

- 0: Standard input (stdin)
- 1: Standard output (stdout)
- 2: Standard error (stderr)

And from my understanding numbers 3 and above are for when you do things like open files and the system assigns it one. With that, let's run `fd` and see what it does:

```bash
fd@ubuntu:~$ ./fd
pass argv[1] a number
```

Alright, let's give it a few number like those standard Linux file descriptors:

```bash
fd@ubuntu:~$ ./fd 0
learn about Linux file IO
fd@ubuntu:~$ ./fd 1
learn about Linux file IO
fd@ubuntu:~$ ./fd 2
learn about Linux file IO
```

Maybe there's more to file descriptors I need to learn about. I should take a look at the [source code](fd.c) for the `fd` program to see :

```bash
fd@ubuntu:~$ vim ./fd.c
```


```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;
	int len = 0;
	len = read(fd, buf, 32);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		setregid(getegid(), getegid());
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}
```

All the programs in C I've written before don't take any arguments to the `main()` function so I'll need to understand what they mean. Maybe arguments from the command line? 

Another thing that sticks out to me is the `char buf[32]`. Since it's outside the scope of the `main()` function, its some kind of global `char` array. I wonder what that does.

Let's doing a quick Google search for `c programming int argc char* argv[]` and see what we can learn. 

This [article](https://www.geeksforgeeks.org/command-line-arguments-in-c-cpp/) from GeeksForGeeks shows that the `int argc` is a number (integer) that holds the quantity of arguments along with the name of the program that you run. The `char* argv[]` argument is an array of `char` pointers which points to an address that contains each part of the command that was executed with the name of the program in position [0].  

As for the `char* envp[]` portion, this [StackOverflow post](https://stackoverflow.com/questions/57009937/what-is-the-const-char-envp-supposed-to-do) shows that it's an pointer to environment variable values. I'm not sure how that ties into it at the moment.

Reading the source code again, it looks like the program checks to see if there are any arguments tacked onto the command and if there isn't, it presents what I saw when I ran the `fd` program by itself. 

On Line 10, I interpret it as taking the 1st argument, converting it to an integer, subtracting it from this hex value `0x1234` from it, and storing it in the `int` variable named `fd`. Converting the hex value gives me the integer value of 4660. 

Putting in vales of 0, 1, 2 that I did earlier would give me -4660 to -4658. Not sure if that means anything at the moment. 

Next up we have a `read()` function. I wonder if there's an `man` page for C functions. Let's run:

```bash
fd@ubuntu:~$ man read
``` 

Okay, now we're getting somewhere! According to the `man` page, it's a system call that reads data from a file descriptor (_Note to self: check if there's `man` pages for C++ functions_). And just my luck, the variable names (`fd` & `bug`) are the same too!

Now Line 10 is making a bit more sense along with that variable declaration on Line 4! Since our standard file descriptors are 0 through 2, I might need to make it so that the variable `fd` is equal to 0 (stdin). So let's head back to the command prompt:

```bash
fd@ubuntu:~$ ./fd 4660

```

Bingo! But it doesn't seem to do anything. At least it's different than it was before. What if I type something into it?
```bash
fd@ubuntu:~$ ./fd 4660
flag
learn about Linux file IO
```

Heading back to the source code might tell me what it's looking for. According to Line 13, it's doing a string comparison with what's in the `buf[]` array and if it matches, then it does some stuff. Let's give "LETMEWIN" a try:

```bash
fd@ubuntu:~$ ./fd 4660
LETMEWIN
good job :)
Mama! Now_I_understand_what_file_descriptors_are!
```

Alright! Let's try entering the line `Mama! Now_I_understand_what_file_descriptors_are!` into the website. And it works!

But I'm curious what the rest of the code is doing. Because we didn't have the permission to `cat` the `flag` file, let's take a look at Line 15. Does that give me permission and how?

```bash
fd@ubuntu:~$ man setregid
```

The `setregid()` function sets the real and/or effective user or group ID of the process that called it. Not sure what the difference between a real vs effective ID is. A quick Google shows this [StackOverflow post](https://stackoverflow.com/questions/32455684/difference-between-real-user-id-effective-user-id-and-saved-user-id) that explains:

>So, the real user id is who you really are (the one who owns the process), and the effective user id is what the operating system looks at to make a decision whether or not you are allowed to do something (most of the time, there are some exceptions).

Since we're talking about permissions, let's take a look at the results of the `ls -alh` command again:

```bash
fd@ubuntu:~$ ls -alh
total 48K
drwxr-x---   5 root fd     4.0K Apr  1 14:50 .
drwxr-xr-x 118 root root   4.0K Jun  1 12:05 ..
d---------   2 root root   4.0K Jun 12  2014 .bash_history
-r-xr-sr-x   1 root fd_pwn  15K Mar 26 13:17 fd
-rw-r--r--   1 root root    452 Mar 26 13:17 fd.c
-r--r-----   1 root fd_pwn   50 Apr  1 06:06 flag
----------   1 root root    128 Oct 26  2016 .gdb_history
dr-xr-xr-x   2 root root   4.0K Dec 19  2016 .irssi
drwxr-xr-x   2 root root   4.0K Oct 23  2016 .pwntools-cache
```

I didn't notice the `s` for the `fd` program in the permissions columns before. Another Google search gives me this post on [StackExchange - Unix & Linux](https://unix.stackexchange.com/questions/118853/what-does-the-s-attribute-in-file-permissions-mean).

The `s` is the marker that tells the operating system to run the `fd` program with the appropriate permission. It looks like using the `setregid()` function sets it up so that the `fd` program runs as the group with the available access (`fd_pwn`) and in turn the `flag` file.

And just out of curiosity, checking the id's for `fd` and `fd_pwn` shows:

```bash
fd@ubuntu:~$ id fd
uid=1002(fd) gid=1002(fd) groups=1002(fd)

fd@ubuntu:~$ id fd_pwn
uid=1006(fd_pwn) gid=1006(fd_pwn) groups=1006(fd_pwn)
```

## Solution
1. Enter an argument of 4660, 4661, or 4662
2. Type in `LETMEWIN`
3. Enter the flag into the site: `Mama! Now_I_understand_what_file_descriptors_are!`

## Lessons

