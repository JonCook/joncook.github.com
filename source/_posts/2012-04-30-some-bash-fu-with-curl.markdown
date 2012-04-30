---
layout: post
title: "Some Bash Fu with Curl, Grep, Awk and Pipes"
date: 2012-04-30 01:15
comments: true
categories: Bash, Curl, Grep, Shell, Scripting
---

I wanted to test out the HTTP response codes from a number of our API endpoints. There is numerous ways to approach this but I thought it would be fun to use the command line and it turned out pretty straight forward.

First off I created a file with some urls in e.g.
```
http://www.google.co.uk
http://www.bbc.co.uk
http://www.arsenal.com
```

And ran the following command:
```
[root@pal ~]# grep "$1" urls.txt | awk '{print "curl --write-out "$0"=http-%{http_code}\"\n\" --silent --output /dev/null "$0'} | sh >> responses.txt
```

This takes each line (url) and passes the argument to curl before outputting the url and http response code for said url into a file called responses.txt
```
http://www.google.co.uk=http-200
http://www.bbc.co.uk=http-200
http://www.arsenal.com=http-302
```

You can then grep to count the number of different http response codes e.g.
```
[root@pal ~]# grep -c http-200 responses.txt
2
```

In reality I ran the following command as wanted to do something a bit more complicated with curl by specifying certs and custom headers
```
grep "$1" urls.txt | awk '{print "curl --cert /etc/pki/my-pem.pem --cacert /etc/my-ca.pem -H \"Accept:application/json\" -H \"X-Candy-Audience:Domestic\" -H \"X-Candy-Platform:HighWeb\" --write-out "$0"=http-%{http_code}\"\n\" --silent --output /dev/null "$0'} | sh >> responses.txt
```

I'm not suggesting this is the best way to do this but nice not to need any frameworks or libraries. If I want to do this more regularly I will probably use a cucumber feature and setup a job in our CI environment not, forgetting a nice looking report but for now this is fine.