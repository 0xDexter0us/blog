+++
title = "Art of Fuzzing and Creating Tailored Wordlist with Scavenger"
date = "2021-11-24T15:45:20+05:30"
author = "Dexter0us"
authorTwitter = "0xDexter0us"
cover = "https://previews.123rf.com/images/paulfleet/paulfleet1603/paulfleet160300007/55229013-old-treasure-map-used-by-pirates-to-find-hidden-treasure.jpg"
tags = ["scavenger", "burp"]
keywords = ["", ""]
description = ""
showFullContent = false

+++

If you have ever watched any interviews or talks of the top bug bounty hunters, you must have noticed one common key point *fuzzing with a target-specific wordlist*, for example hackers hunting for bugs in Google VRP understands the significance of `dogfood`, that single phrase had lead to some of the most critical bugs on internal assets of google but it doesn't have significance in any other program and so there is this fuss about using a custom wordlist but not many resources on how to make one, the best resource I could find was this [talk by TomNomNom](https://www.youtube.com/watch?v=W4_QCSIujQ4) this is a very appreciated talk I highly recommend watching it, creating a wordlist in this way is a tedious task and isn't very efficient as it will contain a lot of noise.

So what can be the efficient way of making a tailored wordlist, let me introduce you to my new tool 

####  [Scavenger](https://github.com/0xDexter0us/Scavenger/) - *burp extension to create a target-specific wordlist from the crawled history.*

{{< image src="/images/scavenger/2021-11-24_20-13.png" position="center" style="border-radius: 5px;" >}}

**How to get started using Scavenger?** It is simple with indispensable prerequisites, primarily you need to set your scope for the target to avoid noise, and then must crawl the target both actively and passively with the burp and must manually click every button and link to populate the history.

**How do Scavenger works?** Scavenger collects three types of lists from the burp history:

**Parameter Wordlist.**
It contains all the unique URL, body, cookie, and JSON parameters, this list can be used for fuzzing for unlinked parameters with tools like Arjun or param-miner, every so often those parameters can contribute to bugs like SSRF and XSS

**Endpoint Wordlist.**
This list is the combination of two lists, first all the endpoints from the URLs of the history, second it extracts all the endpoints from the javascript files (heavily inspired from [GerbenJavado's Link Finder](https://github.com/GerbenJavado/LinkFinder))

**JSON Response Keys Wordlist.**
JSON response keys are the most underrated asset for fuzzing. If an api endpoint `/api/v2/<uuid>/me` returns user data:

```json
{
  "Name": "Test",
  "Mobile": 12345678,
  "Boolean": true,
  "Pets": ["Dog", "cat"],
  "Address": {
    "Permanent address": "USA",
   "current Address": "AU"
  }
}
```



which is secure but what if `/api/v2/<uuid>/Mobile` is not secure and can leak phone number of every user, many times JSON keys can be an unlinked parameter `isAdmin` and `debug` are such an example that everyone is aware of. I use this list for endpoint and parameter fuzzing that's why it deserves to be a separate list.

If you like this project then please star the project on GitHub.
