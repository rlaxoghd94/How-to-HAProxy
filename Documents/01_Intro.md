# Introduction to HAProxy

## What HAProxy is and isn't

HAProxy is:

- a TCP proxy

- an HTTP reverse-proxy(called a "gateway" in HTTP terminology)

- an SSL terminator / initiator / offloader

- a TCP normalizer: A way of protecting fragile TCP stacks from protocol attacks, and also allows to optimize the connection params with the client without having to modify the servers' TCP stack settings

- an HTTP normalizer: When configured to process HTTP traffic, only valid complete requests are passed

- an HTTP fixing tool: it can modify / fix / add / remove / rewrite the URL or any request or response header; which helps fixing interoperability issue in complex environments

- a content-based switch: Content of the request can decide which server to pass the request or connection to. Thus, it's possible to handle multiple protocols over a same port(e.g. HTTP, HTTPS, SSH)

- ***a server load balancer***

- a traffic regulator: it can apply some rate limiting at various points in order to protect the servers against overloading, adjust traffic priorities based on the content, and even pass such info to lower layers and outer network components by marking packets

- a protection against DDoS and service abuse

- an observation point for network troubleshooting

- an HTTP compression offloader

<br></br>

HAProxy is not:

- an explicit HTTP proxy, i.e. the proxy that browsers use to reach the internet

- a caching proxy: it will return the contents received from the server *as-it-is* and will not interfere with any caching policy

- a data scrubber

- a web server

- a packet-based load balanacer: it will not see IP packets nor UDP datagrams, will not perform NAT or even less DSR. These are tasks for lower layers

<br></br>

## How HAProxy works

- HAProxy is a single-threaded, event-driven, non-blocking engine combining a very fast I/O layer with a priority-based scheduler.

- HAProxy is designed with a data forwarding goal in mind and is optimized to move data as fast as possible with the least possible operations

- HAProxy only requires the haproxy executable and a configuration file to run

  - The configuration files are parsed before starting, the HAProxy tries to bind all listening sockets, and refuses to start if anything fails

### What does it do?

Once HAProxy is started, it does exactly 3 things:

- process incoming connections

- periodically check the servers' status(~= health checks)

- exchange information with other haproxy nodes