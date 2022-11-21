---
title: "Unix Sleight of Hand (1/?): Bash Job Control"
date: 2022-11-21
categories:
  - Unix Sleight of Hand
tags:
  - bash
  - sleight of hand
author: Manuel Holtgrewe
images:
  - /posts/2022/2022-11-21-shell-etching.jpg
---

This post deals with the basics "and beyond" of job control in Bash.
The [Bourn Again Shell (Bash)](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) arguably is the most widely used POSIX-compliant shell.
It has been around since 1989 and is still alive and kicking.

{{< figure src="/posts/2022/2022-11-21-shell-etching.jpg" width="50%" caption="'clam shell as an etching' according to [[Stable Diffusion](https://huggingface.co/spaces/stabilityai/stable-diffusion)]." >}}

The core task of a Unix shell is to run programs, so all Linux command line users will be familiar with this.

## Launching Background Jobs

Besides running jobs in the foreground, it is also possible to move jobs to the background.

```bash
$ sleep 1m
# ... block for one minute and return
$ sleep 1m &
[1] 2268
# return immediately
$ ps ax | grep -w 2268
 2268 pts/4    S      0:00 sleep 1m
 2283 pts/4    S+     0:00 grep --color=auto -w 2268
$ jobs
[1]+  Running                 sleep 1m &
$ wait
# block until one minute has passed since the `sleep 1m &` above
```

What is happening here?

1. We run the `sleep` command with parameter `1m` which will wait for one minute until the program returns.
  ("Fun fact") there is a `sleep infinity` that is more robust than `sleep <MANY>d`.
2. We then run the same command and put an ampersand character behind this.
   Bash will interpret this as running the command in the background.
   Bash will print the local job number in square parentheses (ID `1` in this case) and then the PID (process ID) (here `2268`).
3. We look at all currently running processes with `ps ax` and only return lines containing `2268`.
   We get our `sleep` process together with the `grep` that filters the lines.
4. We run the bash **builtin** command `jobs` that tells us that `sleep 1m` is currently running in the background.
5. Finally, we wait for the (actually all) background jobs to return.

This looks already interesting to run a couple of commands in the background.
Is there more?

## Controlling Background Jobs

Of course there is.

First of all, you can move jobs to the background by pressing `Ctrl-Z`:

```bash
$ sleep infinity
^Z
[1]+  Stopped                 sleep infinity
$ jobs
[1]+  Stopped                 sleep infinity
```

Here, we

1. Run `sleep infinity` and then move it to he background by pressing `Ctrl-Z` (which prints as `^Z` on the terminal).
2. Display all running jobs (which happens to only be the one).

We could now continue to move that job to the foreground with `fg`.
Let us launch a second and job first to demonstrate how to handle multiple jobs.

```bash
$ sleep 60m
^Z
[1]+  Stopped                 sleep infinity
$ jobs
[1]-  Stopped                 sleep infinity
[2]+  Stopped                 sleep 60m
$ sleep 30m
^Z
[3]+  Stopped                 sleep 30m
$ jobs
[1]   Stopped                 sleep infinity
[2]-  Stopped                 sleep 60m
[3]+  Stopped                 sleep 30m
```

Now there are three jobs suspended in the background.

We can bring the back with `fg`.

```bash
$ fg 3
sleep 30m
^Z
[3]+  Stopped                 sleep 30m
$ jobs
[1]   Stopped                 sleep infinity
[2]-  Stopped                 sleep 60m
[3]+  Stopped                 sleep 30m
```

In this case, we bring job 3 to the foreground and then suspend it into the background again.

We can tell Bash to unsuspend jobs and let them run in the backgroun with `bg`.

```bash
$ bg 3
[3]+ sleep 30m &
$ jobs
[1]-  Stopped                 sleep infinity
[2]+  Stopped                 sleep 60m
[3]   Running                 sleep 30m &
```

Do you notice how the little plus and minus signs change?
The little plus sign indicates the default for the next job if `fg` or `bg` were called without an argument.
The little minus sign indicates the next default job if the default job was to exit.

We can get the PIDs of the jobs with `job -p`

```bash
$ jobs -p
2655
2662
2669
```

By default, jobs started by the current shell will be terminated once the Bash process ends, for example, when you log out.
You can **detach** the job from the current Bash using the `disown` command.
You can disown the current default job or specify a pid.

```bash
$ disown 2662 
-bash: warning: deleting stopped job 3 with process group 3430
$ ps ax | grep -w 2662
 2662 pts/8    T      0:00 sleep 60m
```

This will detach the stdout and stderr stream from the current Bash and let the job run in the background.
The job will not turn up in the `jobs` output.

```bash
$ jobs -p
2655
2669
```

Note that the job is still suspended (state `T`, see output of `ps ax | grep -2 2662` above).
If you want the job to run, send it the `SIGCONT` signal:

```bash
$ kill -CONT 2662
$ ps ax  grep -w 2662
 2662 pts/8    S      0:00 sleep 30m
```

The state changes to `S` which means sleep/wait for signal - which is what `sleep` does.
A "proper" program would now have `R` there.

You can terminate the program with `kill 2662`.
Note that killing a suspended program is not so easy, you will have to both `kill -KILL` it and then run `kill -CONT` on it so it can actually handle the kill signal.

## Using in Vim and Alternatives

The use of `Ctrl-Z` is useful in interactive editors such as Vim.
For example, you can use it to quickly drop to the Bash for entering a couple of commands on the bash and returning to the vim session.

Alternatively, bash knows the `!` command that allows you to run the current buffer (or selections) through a bash filter.
For example, change to the command mode with `:` and then issue `%!grep foo` will replace the current buffer by lines containing foo.
Similarly, select a couple of lines, enter command mode and you end up with a command line of `'<,'>!grep foo` to achieve the same on the current selection.

Another alternative to get back to the shell is to enter `!bash` as a command.
However, this will create a new shell rather than falling back to the launching shell.
For example, your bash history will be different than from the original Bash process.

{{< figure src="/posts/2022/2022-11-21-confused-cat.jpg" width="50%" caption="'confused cat an etching' according to [[Stable Diffusion](https://huggingface.co/spaces/stabilityai/stable-diffusion)]." >}}

Does this sound confusing?
Don't despair, try it a couple of times.
This does not sound confusing?
Congratulations, you are quite experienced with Linux/bash already.

## What if Ctrl-Z Input is Masked?

Actually, `Ctrl-Z` sends the `PAUSE` signal to the currently running program.
Programs can also **mask** this, in other word prevent the program from being paused and bash sending it to the background.

One example if `lftp`.
Here, your best bet is to execute the Bash from within lftp is to execute a new session with `! bash`.

## Alternatives to Disown

The `nohup` command is a handy alternative to moving jobs to the background and calling `disown` on them.
The following commands will launch two background jobs through the `nohup` command.

```bash
$ nohup sleep infinity &
nohup: ignoring input and appending output to 'nohup.out'
$ nohup sleep infinity >nohup2.out &
nohup: ignoring input and redirecting stderr to stdout
```

By default, nohup will redirect stdout and stderr to `nohup.out`.
The command will mask the `HANGUP` signal to jobs.
If the current Bash session terminates (e.g., if the SSH connection breaks) then the processes launched via `nohup` will continue to run.

If your use case are keeping jobs running on remote hosts, you should better considre tools such as `screen` and `tmux`.

## Further Reading

If you made it so far then thank you for your interest.
Below are a couple of links related to Bash job control.

- ["Job Control Basics" in GNU Bash Manual](https://www.gnu.org/software/bash/manual/html_node/Job-Control-Basics.html)
- ["Jobs and Job Control in Bash Shell" - Baeldung Linux](https://www.baeldung.com/linux/jobs-job-control-bash)
