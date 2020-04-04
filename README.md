# 4190.307 Operating Systems (Spring 2020)
# Project #3: Terminating processes
### Due: 11:59PM (Sunday), April 12

## Introduction

``xv6`` does not provide any way to terminate the currently running process. Hence, when a user runs a process that falls into an infinite loop (e.g., see ``./user/infloop.c``), there is no way to terminate that process. The goal of this project is to implement a mechanism that can terminate one or more processes at once when the ``ctrl-c`` key is pressed in the shell prompt, using the notion of _process group_ introduced in the previous project assignment.

## Background

### Foreground vs. Background processes

A __foreground process__ means a process that is currently interacting with a user. If you run a program in the shell prompt, the corresponding process is the foreground process taking inputs from and/or displaying outputs to your terminal. The shell waits for the completion of such a foreground process by using the ``wait()`` system call. 

You can create a __background process__ as well by appending ``&`` at the end of your command in the shell prompt. Such background processes are run independently without interacting with a user. The shell does not wait for the completion of the background process; the shell simply creates the background process and then take the next input from the user immediately. The following example shows how to run the ``ls`` process in the background on ``xv6``.

```
$ ls &
```

### Terminating processes

In Unix-like operating systems, a user can terminate the execution of the foreground process by simply pressing the ``ctrl-c`` key in the terminal. When the ``ctrl-c`` key is pressed, the kernel sends a signal called ``SIGINT`` to the foreground process and the target process is terminated as soon as it receives the signal (unless the signal is caught by the process).

However, there are a couple of things to consider. First, the foreground process can fork several child processes (e.g., see ``./user/fork10.c``). In this case, all the child processes should be terminated as well. Second, the shell can create more than one processes at once. The following example shows a case where ``cat`` and ``wc`` processes are created and then connected via a pipe. If you look at the process list, you can see that the shell creates another ``sh`` process to manage the whole command; hence the total three processes are created. This means that all the three processes should be terminated together if a user presses the ``ctrl-c`` key.

```
$ cat | wc
^p
1 sleep  init
2 sleep  sh
3 sleep  sh
4 sleep  cat
5 sleep  wc
```

Note that more complicated scenarios are possible, as can be seen below. For more details on how the shell parses and runs the command on ``xv6``, please refer to the ``./user/sh.c`` file.

```
$ cat | cat | cat | cat
^p
1 sleep  init
2 sleep  sh
3 sleep  sh
4 sleep  cat
5 sleep  sh
6 sleep  cat
7 sleep  sh
8 sleep  cat
9 sleep  cat
```
```
$ (cat | wc) | wc
^p
1 sleep  init
2 sleep  sh
3 sleep  sh
4 sleep  sh
5 sleep  wc
6 sleep  cat
7 sleep  wc
```
```
$ (cat; ls) | wc
^p
1 sleep  init
2 sleep  sh
3 sleep  sh
4 sleep  sh
5 sleep  wc
6 sleep  cat
```

### Process groups 

In order to terminate all the related processes together, the Unix-like operating systems use the notion of process groups. The idea is to put all the processes associated with the same job into the same process group and send the ``SIGINT`` signal to all the processes in the given process group.

Consider the following example. We can assign the same PGID (PGID 3 in this example) to all the processes created for the given command. If the kernel knows the PGID of the foreground process group somehow, it can send the ``SIGINT`` signal to all the processes belonging to the process group.

```
$ cat | wc
^p
1 1 sleep  init
2 2 sleep  sh
3 3 sleep  sh
4 3 sleep  cat
5 3 sleep  wc
```

## Problem specification

Your task is to extend ``xv6`` so that a user can terminate the foreground process group by pressing the ``ctrl-c`` key. Because ``xv6`` does not implement any signals, the kernel should kill the corresponding processes directly when the ``ctrl-c`` key is pressed.
In this project, you will need the ``setpgid()`` and ``getpgid()`` system calls implemented in the previous project. 

### 1. Extend the ``kill()`` system call

The ``kill()`` system call in the current ``xv6`` can terminate a single process only. First, you need to extend the ``kill()`` system call (in ``./kernel/proc.c``) so that it can terminate all the processes that have the specified PGID.

__SYNOPSIS__
```
    int kill(int pid);
```

__DESCRIPTION__

The ``kill()`` system call can be used terminate any process group or process.

* If ``pid`` is positive, then the process with the specified ``pid`` is terminated (this is current behavior of the ``kill()`` system call in ``xv6``).
* If ``pid`` is negative, then every process in the process group whose ID (PGID) is ``-pid`` is terminated.
* If ``pid`` is zero, then every process in the process group of the calling process is terminated.

__RETURN VALUE__

* On success (at least one process was terminated), zero is returned. 
* On error (no process was terminated),  -1 is returned.

### 2. Form process groups

You need to assign the same PGID to all the processes created for the given shell command. As mentioned before, there are two cases to consider: (1) when a process forks more than one processes, and (2) when the shell creates multiple processes to implement pipes and a list of commands. For this, you need to modify the shell in ``./user/sh.c``. 

### 3. Make ``ctrl-c`` work

Finally, you need to terminate all the processes belonging to the foreground process group when the ``ctrl-c`` key is pressed by a user. When a key is pressed, it is handled by the kernel's ``consoleintr()`` function in ``./kernel/console.c``. It will be helpful if you see how the list of processes is shown when the ``ctrl-p`` is pressed.

## Skeleton code

You can work on the same code used for the previous project #2. However, as we added two user-level programs, ``infloop.c`` and ``fork10.c`` in the ``./user`` directory, you need to update the skeleton code as follows:

```
$ cd xv6-riscv-snu
$ git pull
$ git checkout pa2
```

If you run the ``ls`` command after booting ``xv6``, you can see that two programs ``infloop`` and ``fork10`` are available as shown below.
```
$ make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 3G -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 1982
cat            2 3 22768
echo           2 4 21624
forktest       2 5 11968
grep           2 6 26112
init           2 7 22344
kill           2 8 21584
ln             2 9 21520
ls             2 10 25024
mkdir          2 11 21680
rm             2 12 21664
sh             2 13 40624
stressfs       2 14 22680
usertests      2 15 121688
wc             2 16 23912
zombie         2 17 21080
setpgid        2 18 21768
getpgid        2 19 21832
infloop        2 20 20928      <------
fork10         2 21 21944      <------
console        3 22 0
$
```

The ``infloop`` program simply enters an infinite loop. Be careful! Once you run the program on ``xv6``, there is no way to stop it other than terminating the whole qemu machine emulator.

The ``fork10`` program is another example that has an infinite loop; it first creates 10 child processes and each process prints its id (from 0 to 9) for every 50 ticks. 

The following example shows what you should get when you implement all the requirements of this project correctly.


```
$ make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 3G -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ infloop
^p
1 1 sleep  init
2 1 sleep  sh
3 3 run    infloop
^c
$ ^p
1 1 sleep  init
2 1 sleep  sh

$ fork10
0
1
2
3
4
5
6
7
8
9
^p
1 1 sleep  init
2 1 sleep  sh
4 4 sleep  fork10
5 4 sleep  fork10
6 4 sleep  fork10
7 4 sleep  fork10
8 4 sleep  fork10
9 4 sleep  fork10
10 4 sleep  fork10
11 4 sleep  fork10
12 4 sleep  fork10
13 4 sleep  fork10
14 4 sleep  fork10
^c
$ ^p
1 1 sleep  init
2 1 sleep  sh

$ cat | cat | cat
^p
1 1 sleep  init
2 1 sleep  sh
16 16 sleep  sh
17 16 sleep  cat
18 16 sleep  sh
19 16 sleep  cat
20 16 sleep  cat
^c
$ ^p
1 1 sleep  init
2 1 sleep  sh
$ fork10 &
$ 0
1
2
3
4
5
6
7
8
9
^p
1 1 sleep  init
2 1 sleep  sh
24 22 sleep  fork10
23 22 sleep  fork10
25 22 sleep  fork10
26 22 sleep  fork10
27 22 sleep  fork10
28 22 sleep  fork10
29 22 sleep  fork10
30 22 sleep  fork10
31 22 sleep  fork10
32 22 sleep  fork10
33 22 runble fork10

$ kill -22
$ ^p
1 1 sleep  init
2 1 sleep  sh
```

The last example (``$ kill -22``) shows that once you implement the ``kill()`` system call as required in this project, it can be also used to kill the backgroud processes using the PGID.

## Restrictions

* Do not add any system calls other than ``setpgid()`` and ``getpgid()`` implemented in the previous project.

## Hand in instructions

* To submit your code, please run ``make submit`` in the ``xv6-riscv-snu`` directory. It will create a file named ``xv6-PA3-STUDENTID.tar.gz`` file in the parent directory. Please upload the file to the server.
* By default, the ``sys.snu.ac.kr`` server is only accessible from the SNU campus network. If you want to access the server outside of the SNU campus, please send a mail to the TA.

## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delay.
* __You can use up to 5 _slip days_ during this semester__. If your submission is delayed by 1 day and if you decided to use 1 slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use in the QnA board of the submission server after each submission. Saving the slip days for later projects is highly recommended!
* Any attempt to copy others' work will result in heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)
