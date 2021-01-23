---
title: "Hacking with ssh one-liners"
date: 2020-05-05
tags: [ssh,hacking]
---

You can run a one-line command in bash on the remote server you ssh to. There
is all sorts of cool things you can do with an ssh client, learn more in
[hacking with ssh](/hacking-with-ssh) post.

```bash
ssh username@hostname [command]
```

The command is passed as an argument to ssh. If the command is anything but a
single word, you are encouraged to wrap it with a single quote character (`'`).
If the username is the same as your current machine username you can omit it.

```bash{outputLines: 2,3}
ssh the-internet 'ls /'
bin
boot
```

That's a very cool feature, considering that you can connect to both the output
*and* the input of the command.

```bash{outputLines: 2,3}
ssh -t the-internet 'read -pecho\>\  x; echo $x'
echo> hello
hello
```

The `-t` option above is necessary to make the `echo>` prompt appear on the
screen. But even without that option, we can still interact with the input of
the `read` command.

## Copying a file over the server

A file can be uploaded to or downloaded from the server easily if we have
access to execute commands.

```bash{outputLines: 4}
echo 'Hello world!' > /tmp/message
cat /tmp/message |ssh the-internet 'cat - > /tmp/remote-message'
ssh the-internet 'cat /tmp/remote-message'
Hello world!
```

## Copying large files fast

With a little help from our friend `gzip` we can even upload/download large
files. We are using a file containing only zeros, and therefore very easy to
zip, but in reality, text files are usually easy to zip as well, therefore this
method is useful.

```bash{outputLines: 2}
ssh the-internet 'du -Dsh /var/tmp/large'
1.0G	/var/tmp/large
```

Now using `gzip` we can zip the file and send the zipped file to `gunzip` on
our local machine to extract:

```bash{outputLines: 2-5}
time ssh the-internet 'gzip --stdout /var/tmp/large' | gunzip > /var/tmp/large

real	0m19.457s
user	0m5.468s
sys	0m0.715s
```

We have the large file in our local machine in no time :tada: :

```bash{outputLines: 2}
du -sh /var/tmp/large
1.1G	/var/tmp/large
```

## Advanced downloading using pv

Suppose you want to see the progress of the file being downloaded, or have more
control the rate of download.

There is a relatively large file on our server called `random`.

```bash{outputLines: 2}
ssh the-internet 'du -Dsh /var/tmp/random'
217M	/var/tmp/random
```

We can see a progress bar when downloading the file (this requires `pv` to be
installed):

```bash{outputLines: 2}
ssh the-internet 'cat /var/tmp/random' | pv -s217m | cat - > random
 103MiB 0:00:14 [7.81MiB/s] [=====>        ] 47% ETA 0:00:15
```

The `-s217m` tells `pv` about the size of the file, if not given the percentage
cannot be calculated.

Or we can limit the download rate to `1 MB` using `-L1m`:

```bash{outputLines: 2}
ssh the-internet 'cat /var/tmp/random' | pv -s217m -L1m | cat - > random
20.0MiB 0:00:20 [1.00MiB/s] [>             ]  9% ETA 0:03:17
```

## Running a command on multiple servers

There are times when we want to run a repatitive command on multiple servers,
we can automate this using ssh. In fact that's how
<a href="https://www.ansible.com/" target="_blank">Ansible</a>
works.

One use-case might be to change password for a user on multiple machines. Let's
change password for user `terminator`:

```bash{outputLines: 2}
echo -e "supersecure\nsupersecure" | ssh the-internet 'sudo passwd terminator'
New password: Retype new password: passwd: password updated successfully
```

We can type in the hosts to change the `terminator` password on:

```bash{outputLines: 3,5}
xargs -IHOST sh -c 'echo "supersecure\nsupersecure" | ssh HOST "sudo passwd terminator"'
the-internet
New password: Retype new password: passwd: password updated successfully
the-skynet
New password: Retype new password: passwd: password updated successfully
```

Or we can automate that and provide the list of hosts:

```text:title=hosts
the-internet
the-skynet
```

```bash{outputLines: 2-3}
xargs -IHOST sh -c 'echo "supersecure\nsupersecure" | ssh HOST "sudo passwd terminator"' < hosts
New password: Retype new password: passwd: password updated successfully
New password: Retype new password: passwd: password updated successfully
```

## Persistent ssh session

Now let's see how we can make use of a combination of ssh and `tmux`. `tmux`
can be used to have an on-going shell in your server that multiple people can
connect to.

To start a session called `mysession`:

```bash{outputLines: 2}
ssh -t the-internet 'tmux new-session -ds mysession'
Connection to the-internet closed.
```

The `-t` tells ssh client to allocate a tty device for using with the tmux
session.

Now anyone with ssh access to that server can connect to the session.

```bash{outputLines: 2,3}
ssh -t the-internet 'tmux attach -t mysession'
[detached (from session mysession)]
Connection to the-internet closed.
```

It is important to detach from the tmux session rather than exit from it if you
want to keep it open for the next time.

You can get a list of clients connected to that session at the moment:

```bash{outputLines: 2,3}
ssh -t the-internet 'tmux list-clients -t mysession'
/dev/pts/1: mysession [150x36 xterm-256color] (utf8)
Connection to the-internet closed.
```

## Connecting to a server using a hop

In case we want to access a third server that is only accessible from a hop
server:

```bash{outputLines: 2}
ssh -tA hop 'ssh the-internet'
u@the-internet:~ $
```

Without `-t` ssh won't allocate tty, the session still works though. When `-A`
is passed the ssh agent is forwarded to the hop, it is only necessary if the
`hop` server doesn't have public key access to `the-internet` host.

There is a better way of doing this using the `-J` option of ssh client, see
the [hacking with ssh](/hacking-with-ssh#connecting-via-a-hop) post for more
info.
