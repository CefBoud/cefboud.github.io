---
title: "It's Not Always DNS: Exploring How Name Resolution Works"
author: cef
date: 2025-11-22
categories: [Technical Writing, Open Source]
tags: [DNS, TCP, C, Networking, Linux]
render_with_liquid: false
description: "An exploration of DNS and Name-to-IP translation. This deep dive explores NSS, getaddrinfo, systemd-resolved and more!"
---

Breaking news: the internet broke last week and DNS wasn't responsible. People refused to believe it at first, but [Cloudflare's blog post](https://blog.cloudflare.com/18-november-2025-outage) left no room for doubt. The meme-sphere, with its repetitive-but-funny "it's always DNS" felt a pang of disappointment. 

![cloudflare outage Poll](/assets/cloudflare-outage.png)

![it was not DNS](/assets/not-dns.png)

The real issue was a ballooning config file that caused a widely deployed program to fail because it couldn't handle files that large.

Anyway, that was the first meaning behind this blog post's title. The second is that there's more to name resolution than DNS alone. It's not *just* DNS.

So, in the spirit of exploring the technologies that make our world work, the ones we rely on every day, this post will take a closer look at DNS and its friends.


## DNS: What it is and why we need it

Computers communicate over the network using IP addresses. To reach a process running on a remote machine (a web server, email server, etc.), you need an IP to get to the computer and a port to identify the service. `172.253.135.139:443` is a web service listening on port 443 (HTTPS) on a machine whose IP is `172.253.135.139`, which happens to be a Google IP.

So if you need to access a service, you need these numbers. But humans don't do well with numbers, we prefer `google.com` to `172.253.135.139`. Using names also allows admins to switch IP addresses without disrupting users.

We need a way to translate (resolve) names to IPs. DNS is one way to do so. The `/etc/hosts` file with name/IP mappings is another. `LDAP` is a third. So when it comes to name resolution, **it's NOT always DNS** (ok, enough silliness).

In the early days of the internet, before DNS, there was a central `hosts.txt` file that contained all computer names and their IPs (there weren't that many). Each host would periodically update its local copy from that central file. This, of course, proved to be unscalable.

The basic function of DNS is simple: you provide it with a domain name, and it provides a Resource Record containing information about that name.



`DomainName Time-to-live Class Type Value`

The domain name is what we're interested in, for example, `cefboud.com` (what a completely random choice!). Time-to-live (TTL) indicates how long a record should be cached. Longer TTLs improve caching and performance, but slow down updates. 
The *Class* is almost always `IN` (internet). The *Type* defines the purpose of the record. *Value* is the actual data stored in the record. Here a record of Type A that provides the IPv4 address of `cefboud.com`, it has a TTL (cache validity) of 86400 seconds (24 hours).

`cefboud.com. 86400 IN A 185.199.109.153`

In most cases, the value is an IP address in the form of an `A` record for IPv4 or an `AAAA` record for IPv6. But many other record types exist.
For example, email uses `MX` records, which indicate the host willing to accept mail for that domain. Aliases (`CNAME`) are another popular type. If there is a CNAME record like:

`hi.cefboud.com. 86400 IN CNAME hello.cefboud.com.`

it means that `hi.cefboud.com` is an alias for `hello.cefboud.com`, and accessing the former is equivalent to accessing the latter.

Another type is `PTR`, used to perform what's called a *reverse lookup*: mapping an IP address back to a name instead of the other way around.

The final type we'll mention is `TXT`, which holds arbitrary text. This can be used, for example, to prove ownership of a domain. In the [ACME protocol](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment) (automatic certificate generation), one challenge requires creating a TXT record with a specific value provided by the certificate issuer. If you publish the TXT record with that value, you prove ownership or control of the domain.



## A DNS Query from Scratch
A piece of code always makes things more real. You can taste the protocol.
Here is a simple Go program that sends a UDP request to 8.8.8.8:53 (Google Nameserver) following the DNS format then parses the result:

```go
func main() {
	if len(os.Args) < 2 {
		fmt.Println("usage: main <domain>")
		return
	}
	domain, dns_server := os.Args[1], "8.8.8.8:53"

	q, id := buildDnsQuery(domain)           // build DNS query
	r, _ := sendAndReceiveUDP(dns_server, q) // send+recv UDP
	parse_response(r, id)                    // print A records
}
func buildDnsQuery(d string) ([]byte, uint16) {
	id := uint16(time.Now().UnixNano())
	h := make([]byte, 12)
	binary.BigEndian.PutUint16(h, id)                            // TXID (random from current timestamp)
	binary.BigEndian.PutUint16(h[2:], 0x100)                     // recursion desired
	binary.BigEndian.PutUint16(h[4:], 1)                         // QDCOUNT=1
	return append(append(h, encodeDomain(d)...), 0, 1, 0, 1), id // QTYPE=1, QCLASS=1
}

func sendAndReceiveUDP(s string, q []byte) ([]byte, error) {
	c, _ := net.Dial("udp", s)
	defer c.Close()
	c.SetDeadline(time.Now().Add(3 * time.Second))
	c.Write(q)
	b := make([]byte, 512)
	n, _ := c.Read(b)
	return b[:n], nil
}
func parse_response(b []byte, id uint16) {
	if binary.BigEndian.Uint16(b) != id {
		return // resposne id different than expected one
	}
	qd, an := int(binary.BigEndian.Uint16(b[4:])), int(binary.BigEndian.Uint16(b[6:]))
	o := 12
	for i := 0; i < qd; i++ {
		_, o = decode_name(b, o)
		o += 4
	} // skip questions
	for i := 0; i < an; i++ { // print A answers
		_, o2 := decode_name(b, o)
		t := binary.BigEndian.Uint16(b[o2:])
		l := int(binary.BigEndian.Uint16(b[o2+8:]))
		r := b[o2+10 : o2+10+l]
		o = o2 + 10 + l
		if t == 1 && len(r) == 4 {
			fmt.Println(net.IP(r))
		}
	}
}
func encodeDomain(d string) []byte { // encode domain labels. Split by '.' then (len + char bytes) and end with 0
	var o []byte
	for _, l := range strings.Split(d, ".") {
		o = append(o, byte(len(l)))
		o = append(o, l...)
	}
	return append(o, 0)
}
// func decode_name(b []byte, o int) (string, int) { // reverse of encodeDomain with support for compression (back references) 

```


```sh
go run main.go cefboud.com 
# 185.199.109.153
# 185.199.110.153
```

What are we doing here? Without getting too much into the weeds of the [DNS message format](https://learn.microsoft.com/en-us/windows-server/networking/dns/message-formats#dns-query-message-header), a DNS packet consists of a header (with a 16-bit transaction ID and 16 bits of flags for various options), followed by questions (basically a domain name and the desired record type: A, AAAA, MX, etc.), and then an answer, which contains the Resource Records we saw above. A respectable protocol.

Since DNS typically uses UDP (falling back to TCP for large packets), it's very simple: you send a stream of bytes containing a domain name and record type (a question), and you get back an answer containing the requested information (usually an IP address).

## Zones

A *zone* is the unit of management in DNS.
Each zone has its own name servers, hosts that store the database of Resource Records (RRs) for that zone. There is no mandatory mapping between a domain and a zone. For example, one could manage `cefboud.com` and all its subdomains *except* for `special.cefboud.com`, which could be managed in its own separate zone. In that case, the `cefboud.com` zone would have an `NS` record delegating authority for that subdomain to the appropriate name servers. This is **DNS delegation**.

Usually there is one *primary* server per zone that stores the data in a file, and a couple of *secondary* servers that pull (replicate) their data from the primary. An "authoritative record" is a record returned by the name server responsible for the domain in question as opposed to a cached record returned by some other DNS server.

DNS is hierarchical.
For `blog.cefboud.com`, the resolution process looks like this:

1. Ask a root server for the name servers of `.com`.
2. Ask the `.com` name servers for the name servers of `cefboud.com`.
3. Ask those servers about `blog.cefboud.com`.

This is called iterative mode.
```
Root (.)
  └── .com NS
        └── cefboud.com NS
              └── blog.cefboud.com (A/AAAA)
```

The other approach is *recursive*: you ask a resolver such as Google's `8.8.8.8`, and it either returns a cached answer or performs all the necessary querying on your behalf.

[Root DNS servers](https://www.iana.org/domains/root/servers) exist and form essential infrastructure for the internet, they're the starting point for all DNS resolution.


If you want an iterative DNS server, it needs to be seeded with the root servers. IANA provides [files](https://www.iana.org/domains/root/files) containing the names and IP addresses of these servers. This is where the Internet's life begins: the Immaculate Internet-ception!


Local DNS servers or popular public ones like Google (`8.8.8.8`) or Cloudflare (`1.1.1.1`) save enormous amounts of traffic through caching. Since most people access many of the same websites, these recursive resolvers perform a lookup once per TTL, and millions of users simply receive it from cache.
One can imagine how this gives these resolvers, or rather, the companies operating them, extremely valuable and sensitive information (basically internet users activity).

In sum, DNS is essentially a massive, worldwide distributed database. Anywhere in the world, you can look up a key (a domain name), specify a record type (`A`, `AAAA`, `MX`, etc.), and get a response.


## The Black Hole that is getaddrinfo()

When exploring DNS (or looking at how name resolution is implemented in many programming languages), you're bound to stumble upon `getaddrinfo()`. This function is powerful enough that it even has its own config file, [`/etc/gai.conf`](https://man7.org/linux/man-pages/man5/gai.conf.5.html), which can be used to change the sorting order of entries returned by `getaddrinfo()` (for example, preferring IPv4 addresses over IPv6).

[From the man page](https://linux.die.net/man/3/getaddrinfo):

> Given node and service, which identify an Internet host and a service, getaddrinfo() returns one or more addrinfo structures, each of which contains an Internet address that can be specified in a call to bind() or connect()

So `getaddrinfo()` takes a host name and a service (e.g., `http`, `ftp`, etc.) and returns something you can pass to `bind()` or `connect()`, meaning something that's directly relevant to networking: an IPv4 or IPv6 address and a port.

There is also a file, `/etc/services`, used by `getaddrinfo()` to translate a human-readable service name to a port number:

```sh
cat /etc/services
# ...
# ftp        21/tcp
# fsp        21/udp       fspd
# ssh        22/tcp                # SSH Remote Login Protocol
# ...
# ntp        123/udp               # Network Time Protocol
# ..
# https      443/tcp               # HTTP protocol over TLS/SSL
# https      443/udp               # HTTP/3
```

Here is a very basic example of using `getaddrinfo`:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <arpa/inet.h>

int main(int argc, char *argv[]) {
    struct addrinfo hints, *res, *p;
    char ipstr[INET6_ADDRSTRLEN];
    memset(&hints, 0, sizeof hints);
    int status = getaddrinfo(argv[1], NULL, &hints, &res);
    if (status != 0) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(status));
        return 2;
    }
    printf("IP addresses for %s:\n\n", argv[1]);

    for (p = res; p != NULL; p = p->ai_next) {
        void *addr;
        char *ipver;

        if (p->ai_family == AF_INET) { // IPv4
            struct sockaddr_in *ipv4 = (struct sockaddr_in *)p->ai_addr;
            addr = &(ipv4->sin_addr);
            ipver = "IPv4";
        } else { // IPv6
            struct sockaddr_in6 *ipv6 = (struct sockaddr_in6 *)p->ai_addr;
            addr = &(ipv6->sin6_addr);
            ipver = "IPv6";
        }

        inet_ntop(p->ai_family, addr, ipstr, sizeof ipstr);
        printf("  %s: %s\n", ipver, ipstr);
    }
    freeaddrinfo(res);
    return 0;
}
```

We provide a domain name as `argv[1]` (the first command-line argument) and pass an `addrinfo` pointer that will hold a linked list of the returned IP addresses (both IPv4 and IPv6). We then traverse the list until we hit `NULL`:

```c
(p = res; p != NULL; p = p->ai_next)
```
and print the results after converting each binary address into human-readable form using `inet_ntop`.

```sh
gcc -o get_ip get_ip.c && ./get_ip cefboud.com
# IP addresses for cefboud.com:
#   IPv6: 2606:50c0:8001::153
#   ...
#   IPv4: 185.199.109.153
```

### Name Service Switch and GlibC
A Linux OS is more than just a kernel, a lot of the heavy lifting is done in the standard C library, most notably glibc. `getaddrinfo` is implemented there. And this is where the "black hole" aspect comes into play.

The implementation relies on the [Name Service Switch](https://en.wikipedia.org/wiki/Name_Service_Switch), or NSS, which handles pluggable, modular name resolution. In `/etc/nsswitch.conf` on my Ubuntu box, I am seeing the following line:

```
hosts:     files dns
```

`getaddrinfo` is just a frontend to these actual name-resolution modules.

This line indicates the lookup order: files (`/etc/hosts`) then `dns`. Another powerful module is `resolve` or `systemd-resolved` which we'll get to in a second. 

Each module has its own shared library:

```sh
ls /lib/aarch64-linux-gnu/
# ..
# libnss_dns.so.2
# libnss_files.so.2
# libnss_systemd.so.2
# ..
```

You can even create your own custom name-resolution library, reference it in `nsswitch.conf` , and have NSS and therefore `getaddrinfo()` and anything that depends on it, use it.

The `dns` module in `nsswitch.conf` corresponds to `libnss_dns`, which is implemented in glibc and relies on `/etc/resolv.conf` to determine which nameserver to send DNS requests to:

```sh
cat /etc/resolv.conf
nameserver 127.0.0.53
```

127.0.0.53?? That's a local address. Very intriguing! We'll return to this mystery in a moment.

So `libnss_dns` sends a DNS query to the configured nameserver, parses the result, and returns it to `getaddrinfo`. But aside from that, not much magic happens and crucially, there is no caching.

## systemd-resolve

We saw `resolve` above and also noticed a `libnss_systemd` library in our GNU libraries folder. These refer to a systemd service responsible for name resolution. It handles a lot of other DNS features (DNSSEC, multicast DNS for small devices, and others). The nameserver configured earlier as `127.0.0.53` is actually systemd-resolved acting as a *DNS stub*, taking care of communicating with external nameservers. And importantly, it implements caching!

`resolvectl` is used to interact with systemd-resolved. We can check the cache, view statistics, and configure upstream nameservers.

```sh
resolvectl show-cache
# ...
# google.com IN A 172.253.135.100
# cefboud.com IN A 185.199.109.153
# ...

resolvectl statistics
# ...                                              
# Cache                                          
#                          Current Cache Size:  7
#                                  Cache Hits:  2
#                                Cache Misses: 17
# ...

# empty the cache (cache is in memory, so restarting also clears it)
resolvectl flush-caches
```

Every time I run `nslookup <domain>` or any command that triggers name resolution (`curl`, `ping`, etc.), I can see the cache statistics change: a +1 or +2 in cache size and cache misses if it's the first time I've queried that domain, or a +1/+2 in cache hits if the domain is already cached.

One other interesting thing is that the default nameservers used by Linux systems running `systemd-resolved` are specified [here](https://github.com/systemd/systemd/blob/f295cfa1a758147226308f802c19a12fd1a95715/meson_options.txt#L377):

```
value : '1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google 9.9.9.9#dns.quad9.net 1.0.0.1#cloudflare-dns.com 8.8.4.4#dns.google 149.112.112.112#dns.quad9.net 2606:4700:4700::1111#cloudflare-dns.com 2001:4860:4860::8888#dns.google 2620:fe::fe#dns.quad9.net 2606:4700:4700::1001#cloudflare-dns.com 2001:4860:4860::8844#dns.google 2620:fe::9#dns.quad9.net'
```

I believe these are fallback values and are used only when no DNS servers are provided by the local network (DHCP) or via `resolv.conf`. Still, it's something to think about, how much value and power a default can carry!

## Making sure it's not all bogus

Is all of this bogus? How can we be sure the docs and the code aren't lying?
Let's run `strace` and print all the syscalls used when pinging yours truly's website.


```sh
strace -o /tmp/strace-ping ping cefboud.com

cat /tmp/strace-ping  | grep "/etc/"

# openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
# newfstatat(AT_FDCWD, "/etc/nsswitch.conf", {st_mode=S_IFREG|0644, st_size=526, ...}, 0) = 0
# openat(AT_FDCWD, "/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 5
# read(5, "# /etc/nsswitch.conf\n#\n# Example"..., 4096) = 526
# newfstatat(AT_FDCWD, "/etc/resolv.conf", {st_mode=S_IFREG|0644, st_size=920, ...}, 0) = 0
# openat(AT_FDCWD, "/etc/host.conf", O_RDONLY|O_CLOEXEC) = 5
# openat(AT_FDCWD, "/etc/resolv.conf", O_RDONLY|O_CLOEXEC) = 5
# openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 5
# openat(AT_FDCWD, "/etc/gai.conf", O_RDONLY|O_CLOEXEC) = 5
# newfstatat(AT_FDCWD, "/etc/resolv.conf", {st_mode=S_IFREG|0644, st_size=920, ...}, 0) = 0
# newfstatat(AT_FDCWD, "/etc/nsswitch.conf", {st_mode=S_IFREG|0644, st_size=526, ...}, 0) = 0
# openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 5
```


Isn't this beautiful? We see it all: `nsswitch.conf` used by NSS, `resolv.conf` used by glibc's `libnss_dns` module, `gai.conf` used by `getaddrinfo()`, and even `/etc/hosts`, which is read by the `files` module specified in `nsswitch.conf`.



<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/MHxTeJEJQjY"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>