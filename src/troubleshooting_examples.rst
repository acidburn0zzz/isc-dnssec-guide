Troubleshooting Example
=======================

Let's put what we've looked at together into an example and see the
steps taken to solve the problem. We start with someone complaining that
she is unable to resolve the name ``www.example.com``. We use dig on her
machine to verify the behavior, and we received the following output
from ``dig``:

::

   $ dig www.example.com. A

   ; <<>> DiG 9.10.1 <<>> www.example.com. A
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 26068
   ;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;www.example.com.       IN  A

   ;; Query time: 784 msec
   ;; SERVER: 192.168.1.7#53(192.168.1.7)
   ;; WHEN: Mon Nov 03 20:00:45 CST 2014
   ;; MSG SIZE  rcvd: 44

We learned from this output that the recursive name server 192.168.1.7
returned a generic error message when resolving the name
``www.example.com``. The next step is to look at the DNS server
configuration on 192.168.1.7 to see how it is configured. Below is an
excerpt of ``named.conf`` from 192.168.1.7:

::

   options {
       ...
       forwarders {192.168.1.11;};
       forward only;
       ...
   };

This tells us that the recursive name server 192.168.1.7 just sends all
recursive queries to 192.168.1.11. Let's query 192.168.1.11:

::

   $ dig @192.168.1.11 www.example.com. A

   ; <<>> DiG 9.10.1 <<>> @192.168.1.11 www.example.com. A
   ; (1 server found)
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 24171
   ;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
   ...

And we get the same result as when we queries 192.168.1.7, generic
failure message, but we also learned that 192.168.1.11 is not
authoritative for ``example.com`` (no ``aa`` flag), so it is getting
this response from somewhere else. Below is the configuration excerpt
from 192.168.1.11:

::

   options {
       ...
       forwarders {};
       forward only;
       ...
   };

   zone "example.com" IN {
       type forward;
       forwarders { 192.168.1.13; };
       forward only;
   };

At first glance, it may look like 192.168.1.11 is just performing
recursion itself, querying Internet name servers directly; however,
further down the configuration file, we see the forward zone definition,
which tell us that 192.168.1.11 is doing conditional forwarding just for
``example.com``, and it is sending all example.com queries to
192.168.1.13.

We then query 192.168.1.13:

::

   $ dig @192.168.1.13 www.example.com. A

   ; <<>> DiG 9.10.1 <<>> @192.168.1.13 www.example.com. A
   ; (1 server found)
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35962
   ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
   ;; WARNING: recursion requested but not available

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;www.example.com.       IN  A

   ;; ANSWER SECTION:
   www.example.com.    600 IN  A   192.168.1.100

   ;; Query time: 4 msec
   ;; SERVER: 192.168.1.13#53(192.168.1.13)
   ;; WHEN: Mon Nov 03 20:06:26 CST 2014
   ;; MSG SIZE  rcvd: 60

Finally! We found the authoritative name server! Now we know our query
path looks like this:

But 192.168.1.13 has no trouble answering the query for
``www.example.com``, so the problem might be between 192.168.1.11 and
192.168.1.13? We know there are no firewalls or network devices between
192.168.1.11 and 192.168.1.13 that could intercept packets. Let's query
192.168.1.11 again, but this time, let's purposely turn off DNSSEC
validation by using ``+cd`` (checking disabled), to see if this error
message was caused by DNSSEC validation:

::

   $ dig @192.168.1.11 www.example.com. A +cd

   ; <<>> DiG 9.10.1 <<>> @192.168.1.11 www.example.com. A +cd
   ; (1 server found)
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58332
   ;; flags: qr rd ra cd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;www.example.com.       IN  A

   ;; ANSWER SECTION:
   www.example.com.    562 IN  A   192.168.1.100

   ;; Query time: 2 msec
   ;; SERVER: 192.168.1.11#53(192.168.1.11)
   ;; WHEN: Mon Nov 03 20:01:23 CST 2014
   ;; MSG SIZE  rcvd: 60

Bingo! So the problem is on 192.168.1.11, and specifically, with DNSSEC
validation. Now we can focus our attention on the configuration on
192.168.1.11, examine its logs, check its system time, or check its
trust anchors, to see what may be the root cause.

Examining log messages from 192.168.1.11, we notice the following two
entries:

::

   error (no valid KEY) resolving 'example.com/DNSKEY/IN': 192.168.1.13#53
   error (broken trust chain) resolving 'www.example.com/A/IN': 192.168.1.13#53

So it would appear that on the server 192.168.1.11, there is a broken
trust chain. At this point, we can probably conclude the problem is in
one of the trusted-keys statements on 192.168.1.11, but let's turn on
DNSSEC debug logging (as described in
`??? <#troubleshooting-logging-debug>`__), and re-run the ``dig`` for
``www.example.com`` one more time to see what log messages get
generated:

::

   ...
   validating @0xb4b48968: example.com DNSKEY: attempting positive response validation
   validating @0xb4b48968: example.com DNSKEY: unable to find a DNSKEY which verifies the DNSKEY RRset and also matches a trusted key for 'example.com'
   validating @0xb4b48968: example.com DNSKEY: please check the 'trusted-keys' for 'example.com' in named.conf.
   ...

Okay, so we have a confirmed log message telling us to look at
'``trusted-keys``'. The ``named.conf`` on 192.168.1.11 contains the
following:

::

   trusted-keys {
       example.com. 257 3 8 "AwEAAbluLK0k3dPKnsJNd5tGbO5bgh7WuXzaSDQVwi/qqPdCR65ZDiin
                             0GTpL++B1iKYDP4rRL/s/2TMppI1fV638f2SuhNQ9zYIuCo/FuHeJB7/
                             DBQ03eJFvN1QHC0we2uUFrXazz8eT9nkI1SUu0fhcs6CA06gGqauDbpU
                             mpM7VUX3";
   };

Let's check the authoritative server (192.168.1.13) for the correct key:

::

   $ dig @192.168.1.13 example.com. DNSKEY +multiline

   ; <<>> DiG 9.10.1 <<>> @192.168.1.13 example.com. DNSKEY +multiline
   ; (1 server found)
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38451
   ;; flags: qr aa rd; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1
   ;; WARNING: recursion requested but not available

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 4096
   ;; QUESTION SECTION:
   ;example.com.       IN DNSKEY

   ;; ANSWER SECTION:
   example.com.        600 IN DNSKEY 256 3 8 (
                   AwEAAbluLK0k3dPKnsJNd5tGbO5bgh7WuXzaSDQVwi/q
                   qPdCR65ZDiin0GTpL++B1iKYDP4rRL/s/2TMppI1fV63
                   8f2SuhNQ9zYIuCo/FuHeJB7/DBQ03eJFvN1QHC0we2uU
                   FrXazz8eT9nkI1SUu0fhcs6CA06gGqauDbpUmpM7VUX3
                   ) ; ZSK; alg = RSASHA256; key id = 4974
   example.com.        600 IN DNSKEY 257 3 8 (
                   AwEAAb4N53kPbdRTAwvJT8OYVeVhQIldwppMy7KBJ+8k
                   Uggx2PU3yP/qlq4Zjl0MMmqRiJhD/S+z9cJLNTZ9tHz1
                   7aZQjFyGAyuU3DGW16xfMolcIn+c8TpPCzBOFhxk6jvO
                   VLlz+Wgyi1ES+t29FjYYv5cVNRPmxXLRjlHFdO1DzX3N
                   dmcUoZ+VVJCvaML9+6UpL/6jitNsoU8JHnxT9B2CGKcw
                   N7VaK4l9Ida2BqY3/4UVqWzhj03/M5LK6cn1pEQbQMtY
                   R0TNJURBKdK8bH663h98i23tVX0/85IsCVBL4Dd2boa3
                   /7HPp7uZN1AjDvcRsOh1mqixwUGmVm1EskDIMy8=
                   ) ; KSK; alg = RSASHA256; key id = 45319
   example.com.        600 IN DNSKEY 256 3 8 (
                   AwEAAfbc/0ESumm1mPVkm025PfHKHNYW62yx0wyLN5LE
                   4DifN6FzIVSKSGdMOdq+z6vFGxzzjPDz7QZdeC6ttIUA
                   Bo4tG7dDrsWK+tG5cm4vuylsEVbnnW5i+gFG/02+RYmZ
                   ZT9AobXB5bVjfXl9SDBgpBluB35WUCAnK9WkRRUS08lf
                   ) ; ZSK; alg = RSASHA256; key id = 60798
   example.com.        600 IN DNSKEY 257 3 8 (
                   AwEAAb3lVweaj4dA9dvmcwlkaVpJ4/3ccXbRjgV7jqh1
                   p0REL8fI0Z42E9SdxdsdTi+2XYcmHDQYEoqwYh70t/4P
                   4oObZFIUHl+hhKLdXQNZGtzT0xF60k527N9cHPddoXzg
                   AXYBtGLlLMSJcV8s0rw/i+64xNGdRWpFRdo78RhJ5LU3
                   1SAPUnhi3OvJgsOpBPntrSyX6iA5ZotitxZJNTqP+Jck
                   lhPWFgFOBgdvWJ369BRlDGy/m8+pctypZq1hy7ZteHet
                   r55/cLBXY1BEzz3Q8vLUnSOu5An8IF0v2Gt7hOyY3nqu
                   bU5vjCbogLj1K5ySBAJbHcCPAFrPGSIfmRize+U=
                   ) ; KSK; alg = RSASHA256; key id = 40327

   ;; Query time: 4 msec
   ;; SERVER: 192.168.1.13#53(192.168.1.13)
   ;; WHEN: Mon Nov 03 21:51:28 CST 2014
   ;; MSG SIZE  rcvd: 888

Did you spot the mistake? We have the correct key data in our
configuration, but the key type was incorrect. In our configuration, the
key was configured as a KSK (257), while the authoritative server
indicates that it is a ZSK (256).
