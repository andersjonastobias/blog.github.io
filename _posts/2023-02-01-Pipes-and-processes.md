---
layout: post
title: "Pipes and processes"
comments_id: 4
date: 2023-02-01
tags: [Python, Linux, bash]
comments: 
---


### Processes, pipes and filedescriptors

As we saw in an earlier entry it can at times be convenient to create subprocesses and make them talk to each other. In the bash-shell this is easily done since basically any command in bash will spawn a new process (to find these relevant commands look in `/sbin`, `/bin`, `/local/bin` or `/home/local/bin` - or use `compgen -ac |sort` ). In this post we will try to spawn some processes from within python and make them talk to each other.  Note that this note draws very heavily on inspiration form the following sources
1. https://linuxhint.com/python-subprocess-pipes/
1. https://lyceum-allotments.github.io/2017/03/python-and-pipes-part-5-subprocesses-and-pipes/
1. https://www.geeksforgeeks.org/python-os-mkfifo-method/


### Spawning subprocesses
Let us start with this command from the shell
``` 
ls |  grep "temp" ; ps -A -f > "templog.txt"
```
This shell will spawn a total of three subprocesses, as can be checked by running it repeatedly and checking that the PID of ps always jumps by three. 

Let us make python carry out the similar task
```Python
import subprocess

p1 = subprocess.Popen(["ls"], stdout=subprocess.PIPE)
fd = open("templog2.txt",'wb')
p2 = subprocess.run(["grep", "temp"], stdin=p1.stdout, stdout=fd)
p3 = subprocess.run(["ps","-A","-f"], stdin=p1.stdout, stdout=fd)


```
Note that in order to make this script work we need to run python from a bash shell in ubuntu (or wsl) since the command above will delegate to the operating system, and hence the relevant shell-commands used above need to work.

We see that by setting `subprocess.PIPE` in our first script we can get the stdout of the process, and feed this into the next process. Similarly, we could have fed stdout into a file we have opened. Opening a file will return an `io.TextIOWrapper` from the library `io`. This is the same type of object as `sys.stdin` or `sys.stdout` in the standard library. Note that such a `TextIOWrapper` has a `fileno()`-method which will return the filedescriptor.  The interesting thing here is that if we look at `p1.stdout` this is not a `TextIOWrapper` but a `BufferedReader` in `io`. To understand the relation between these to, we can import the inspect-module, and look at the relevant ancestors
```
>>> inspect.getmro(io.BufferedReader)
(<class '_io.BufferedReader'>, <class '_io._BufferedIOBase'>, <class '_io._IOBase'>, <class 'object'>)
>>> inspect.getmro(io.TextIOWrapper)
(<class '_io.TextIOWrapper'>, <class '_io._TextIOBase'>, <class '_io._IOBase'>, <class 'object'>)
```
Hence we see that they share that class `io.IOBase`, and sure enough this class holds the `filno()`-metod.
```
>>> dir(io.IOBase)
['__abstractmethods__', '__class__', '__del__', '__delattr__', '__dict__', '__dir__', '__doc__', '__enter__', '__eq__', '__exit__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__iter__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__next__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '_abc_impl', '_checkClosed', '_checkReadable', '_checkSeekable', '_checkWritable', 'close', 'closed', 'fileno', 'flush', 'isatty', 'readable', 'readline', 'readlines', 'seek', 'seekable', 'tell', 'truncate', 'writable', 'writelines']
```

So even though `sys.stdout` and the version of `stdout` used when piping are not the same type, they do share the same ancestor, and hence some key methods.

Another thing to notice is that our stdin and stdout are streams, hence they can be read once, and are then empty
```
>>> a = subprocess.Popen(["ls"], stdout=subprocess.PIPE)
>>> a.stdout.read()
b'Processes, pipes and filedescriptors.md\nVilhelm.md\nVilhelm.pdf\n__pycache__\ndel2\ndel2.zip\nimage.png\nimage2.png\nimage3.png\nimage5.png\nimage6.png\nimage7.png\nmarkdown\nmarkdown.zip\nminesweeper.py\nnoter,del2.md\nnoter,del2.pdf\nscript.py\nsnip1.png\nsnip2.png\nsnip3.png\nsnip4.png\nsnip5.png\nsnip6.png\nsnip7.png\nstackoverflow question log\nstackoverflow question.py\ntemplog2.txt\ntest.py\ntest1.py\nvscript.py\n'
>>> a.stdout.read()
b''
```

### Making processes talk to stdin
To make our processes run programs, and interact with these let us make the following simple script

```python
import sys
print ("Name?")
for name in iter(sys.stdin.readline, ''):
    name = name[:-1]
    if name == "exit":
        break
    print ("Your name is {0}?".format(name))
    print ("\n Name?")
```
Using the simple script
```python
import subprocess
import sys

proc = subprocess.Popen(["python3", "script.py"])
while proc.returncode is None:
    proc.poll()

proc = subprocess.Popen(["python3", "script.py"],
                        stdin=subprocess.PIPE, stdout=subprocess.PIPE)
proc.stdin.write(b"AAA\n")
proc.stdin.write(b"BBBB\n")
proc.stdin.close()
while proc.returncode is None:
    proc.poll()

print("I am back from the child program this:\n{0}".format(proc.stdout.read()))
```
this program will start a subprocess and write to stdin and stdout of this subprocess. 

### Pipes and filedescriptors
Let us delve a bit more into the details of these pipes and how they work. 

Essentially a pipe is a bridge between two processes, and they can be created by the operating system
```
>>> os.pipe()
(8, 9)
```
what is returned is the filedescriptors for resp. the read and the write-end of the pipe. Let us try to write to one end, and then read from the other.
```
>>> os.write(9,b"test")
4
>>> os.read(8,10)
b'test'
```
Trying to write to the wrong end will cause an error so the two end of our pipe is one-way each
```
>>> os.write(8,b"test")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
OSError: [Errno 9] Bad file descriptor
```

Note that these pipes we create are global objects maintained on the operating-system level. They are in many ways comparable to files  as can be seen here
```python
# Python program to explain os.pipe() method

# importing os module
import os


# Create a pipe
r, w = os.pipe()

# The returned file descriptor r and w
# can be used for reading and
# writing respectively.

# We will create a child process
# and using these file descriptor
# the parent process will write
# some text and child process will
# read the text written by the parent process

# Create a child process
pid = os.fork()

# pid greater than 0 represents
# the parent process
if pid > 0:

	# This is the parent process
	# Closes file descriptor r
	os.close(r)

	# Write some text to file descriptor w
	print("Parent process is writing")
	text = b"Hello child process"
	os.write(w, text)
	print("Written text:", text.decode())

	
else:

	# This is the parent process
	# Closes file descriptor w
	os.close(w)

	# Read the text written by parent process
	print("\nChild Process is reading")
	r = os.fdopen(r)
	print("Read text:", r.read())
```

To list the pipes used by a given process start by determining the process-id
```
ps -A -f
```
After this we go to the directory `/proc/<pid>/fd`, and list with full details 
```
anders@DESKTOP-439E2GN:/proc/251/fd$ ls -l
total 0
lrwx------ 1 anders anders 64 Jan 31 15:20 0 -> /dev/pts/2
lrwx------ 1 anders anders 64 Jan 31 15:20 1 -> /dev/pts/2
lrwx------ 1 anders anders 64 Jan 31 15:20 2 -> /dev/pts/2
lr-x------ 1 anders anders 64 Jan 31 15:20 3 -> 'pipe:[18913]'
l-wx------ 1 anders anders 64 Jan 31 15:20 4 -> 'pipe:[18913]'
l-wx------ 1 anders anders 64 Jan 31 15:22 5 -> /mnt/c/Users/Bruger/templog2.txt
```
Here we see that we have standard-input, standard-output and standard-error linked to our standard device, two additional file-descriptors linked to a pipe, and finally, we have a file opened. We also see in the left-most column that one end of the pipe is opened to reading whereas the other is opened to writing (in this connection it is a bit surprising that standard-input appears to be open for writing...).

To list pipes across all processes used 
```
lsof | grep pipe
```
which will list all open files (where files are understood in a very general sense to inlude pipes, sockets, etc.).
