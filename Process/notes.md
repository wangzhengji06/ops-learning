## General Idea

How does process run?

Memory created -> Add PID -> initilize the process  -> Process ready to be run

Where does the process run?

In CPU, by scheduler, with time split unit

How is process managed?

Nice value, the lower the more important, get more consistent session and higher frequency.

Every process is forked by a parent process, the first process is init/systemd, use `pstree` to confirm.



### Process vs Thread

Process does not share the same memory, while the thread share the memory of process

ls /proc/{PID} to check the process and thread


### IPC

inter-process communication 

pipe / port / socket


### Priority

nice -n / renice -n

## Process Management

### Check 

pstree -p | grep xx

ps  -a : processes in all terminals
    -u : show all process owners uid
    -x : even if it is not related to terminal, still show 
    -f : forest view
    -eo: customized the fields you want to show

You mostly only need ps auxf

vmstat: Check what part goes wrong
       -n -1: real time update

top: Check what program causes the problem, then you can go check the log
    -P: ranked based on CPU usage
    -M: ranked based on Memory usage
    -N: ranked based on process id
    -T: ranked based on running time
    -k: sigterm a process
    -n: renice a process

iostat: for I/O

lsof -Pti [ip]:[port]: Check processes related to ports

lsof [file]: check which process is using this file

#### memory 

cache: Read source code for memory

buffer: write source code for memory

available = total - used, becuase all the other sections can throw away the data.

echo 3{1,2} > /proc/sys/vm/drop_caches


### Manipulate

kill -0 pid: is this process still alive?

kill -9 pid: kill this process completely

killall program: kill all processes realted to this program

pkill: same as killall

nohup: detach from this terminal session

&: run in background, noticed that this is also you run several command simultaneously

a typical example is: nohup sleep 1000 > /dev/null 2&>1 &

ctrl + z: foreground to background, and stopped it

fg %Pid:: send it back to foreground



## Task Management 

Two types: one-shot / recurrent

Both of these tasks need daemon service: atd / crond

at: at xx to interatively scheudle a future task

at -l: check the at job list

crontab:

Min Hour Day Month DOW

\* all values

, seperate individual values

\- range of values

/ divide a value into steps

crontab -l: check cron tasks list
