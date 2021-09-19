+++
title = "Go Blogs Hacktivitycon 2021 Writeup [Golang SSTI]"
date = "2021-09-19T13:18:59+05:30"
author = ""
authorTwitter = "" #do not include @
cover = "https://miro.medium.com/max/788/0*wj44yAjifjt6VlCP.png"
tags = ["ctf", "hacktivitycon", "hackerone", "golang", "ssti"]

showFullContent = false

+++

This was my first ever jeopardy style CTF and for most my team mates as well, I was kind of lost after seeing so many challenges then I saw this tweet from [John Hammond](https://twitter.com/_JohnHammond) and I took it as a challenge to solve it. 

{{< image src="/images/go-blogs/2021-09-19_14-17.png" position="center" style="border-radius: 5px;" >}}





So I started the challenge with the basic enumeration, directory fuzzing. It was a simple blog writing application made in golang, After registering an account and logging in you'll be greeted with this home page, and functionalities to add new post and edit username in profile page. In directory bruteforcing I discovered `/admin` and `/models` initially. `/admin` was the page where we will find the flag, and `/models` directory contained the source code.

{{< image src="/images/go-blogs/2021-09-19_14-28.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/go-blogs/2021-09-19_14-28_1.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/go-blogs/2021-09-19_14-30.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/go-blogs/2021-09-19_14-20.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/go-blogs/2021-09-19_14-21_1.png" position="center" style="border-radius: 5px;" >}}



While doing the initial testing I discovered the XSS in the posts, so my thought process went the direction of blind XSS, so I tested for blind XSS and the only request I received was from my own browser. 

{{< image src="/images/go-blogs/2021-09-19_14-29.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/go-blogs/2021-09-19_14-40.png" position="center" style="border-radius: 5px;" >}}

Then I moved forward to do static analysis and read the golang code, the code was well written in terms of paramterized query and everything, the biggest hint was infront of my eyes and I missed that, meanwhile I had spent more than 3-4 hours on it till now.
Then I tried to look deeper and started to fuzz directories to discover if I am missing something or not, then `web` directory popped up containing templates of the blog. I think most of the hacker would have missed these two `models` and `web` directories as when you visit `web` or `models` without a trailing backslash `/` it will give you 404 error.

{{< image src="/images/go-blogs/2021-09-19_14-21.png" position="center" style="border-radius: 5px;" >}}



After I read the template files from `/web` directory then dots started to connect for me, I was like HOLY COW this was SSTI and how did I missed it while reading the source code. Thankfully I had already read the research by **Gus Ralph** of onsecurity.io  [Method Confusion In Go SSTIs Lead To File Read And RCE.](https://www.onsecurity.io/blog/go-ssti-method-research/) and I had some very basic idea of SSTI in golang but SSTI in golang is very less researched till now, you can find only two posts about it on the internet.

```url
https://www.onsecurity.io/blog/go-ssti-method-research/
https://blog.takemyhand.xyz/2020/05/ssti-breaking-gos-template-engine-to.html
```

SSTI in Go isn't as simple as sending `{{8+8}}` and checking for `16` in the source code, as templates in golang is much different from other templating languages like Jijna2, Twig, in those languages for example we use `{{ pageTitle }}` but in golang we use `.` parameter  like this `{{ .pageTitle }}`

From this point my team mate [monke](https://twitter.com/pmofcats) joined me on this challenge.
So we started testing the SSTI with `{{ . }}` payload, and we discovered that POST request to `/profile` endpoint is vulnerable here. Which outputs the data struct being passed as input to the template.

{{< image src="/images/go-blogs/2021-09-19_15-26.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/go-blogs/2021-09-19_15-27.png" position="center" style="border-radius: 5px;" >}}

We tried some system commands to test our scope here but it was not going to be that easy here. We then tried to impersonate the admin account with payload  `{{.Post.Author.Username}}`, and when we visited `/admin` endpoint it didn't worked.

At this point we were stuck here for long time and then [pmnh](https://twitter.com/h1pmnh) came for our rescue, he read the source code thoroughly and found an interesting function `ChangePassword` which required us to pass the user struct with the password. In this way we were able to change the admin password.

{{< image src="/images/go-blogs/2021-09-19_15-31.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/go-blogs/2021-09-19_17-18.png" position="center" style="border-radius: 5px;" >}}

So first we had to set payload to `{{ .Post.Author }}`
Then we will have to update the password of the Author by `{{ .Post.Author.ChangePassword "123456" }}`

Now you have successfully changed the password of admin `congo@congon4tor.com` to `123456`.
Next step was to login and grab the flag.

{{< image src="/images/go-blogs/2021-09-19_16-43.png" position="center" style="border-radius: 5px;" >}}

We were the second team to solve it out of all ten solves by the end, just after [optionalCTF](https://twitter.com/optionalctf).

{{< image src="/images/go-blogs/2021-09-19_14-15.png" position="center" style="border-radius: 5px;" >}}

Me and my team mates enjoyed solving this challenge and the whole CTF thoroughly, As a team of all new guys, playing for the first time in a CTF together we ended up at 57th position out of 2527 teams with 5049 points.

{{< image src="/images/go-blogs/points.png" position="center" style="border-radius: 5px;" >}}



Shout-out to [@John Hammond](https://twitter.com/_JohnHammond) [@Congon4tor](https://twitter.com/Congon4tor) for amazing CTF and all the help, and my amazing [team mates.](https://ctf.hacktivitycon.com/teams/61) 
