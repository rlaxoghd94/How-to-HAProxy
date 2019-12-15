# Configuration

## Initial Configuration

1. Review the default configuration file at `/etc/haproxy/haproxy.cfg`, which is created automatically during installation. This file defines a standard setup without any load balancing:

```s
# /etc/haproxy/haproxy.cfg

global
  log /dev/log    local0
  log /dev/log    local1 notice
  chroot /var/lib/haproxy
  stats socket /run/haproxy/admin.sock mode 660 level admin
  stats timeout 30s
  user haproxy
  group haproxy
  daemon

  # Default SSL material locations
  ca-base /etc/ssl/certs
  crt-base /etc/ssl/private

  # Default ciphers to use on SSL-enabled listening sockets.
  # For more information, see ciphers(1SSL). This list is from:
  #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
  # An alternative list with additional directives can be obtained from
  #  https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
  ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
  ssl-default-bind-options no-sslv3

defaults
  log     global
  mode    http
  option  httplog
  option  dontlognull
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  errorfile 400 /etc/haproxy/errors/400.http
  errorfile 403 /etc/haproxy/errors/403.http
  errorfile 408 /etc/haproxy/errors/408.http
  errorfile 500 /etc/haproxy/errors/500.http
  errorfile 502 /etc/haproxy/errors/502.http
  errorfile 503 /etc/haproxy/errors/503.http
  errorfile 504 /etc/haproxy/errors/504.http
```

The `global` section defines system level parameters such as file locations and the user and group under which HAProxy is executed. In most cases you will not need to change anything in this section. The user **haproxy** and group **haproxy** are both created during installation

The `defaults` section defines additional logging parameters and options related to timeouts and errors. By default, both normal and error messages will be logged

2. If you wish to disable normal operation messages from being logged you can add the following after `option dontlognull`:

```s
option dontlog-normal
```

3. You can also choose to have the error logs in a separate log file:

```s
option log-separate-errors
```

## Configure Load Balancing

When you configure load balancing using HAProxy, there are two types of nodes which need to be defined: **frontend** and **backend**. The frontend is the node by which HAProxy listens for connections. Backend nodes are those by which HAProxy can forward requests. A third node type, the stats node, can be used to monitor the load balancer and the other two nodes.

1. Open `/etc/haproxy/haproxy.cfg` in a text editor and append the configuration for the front-end:

```s
# /etc/haproxy/haproxy.cfg
frontend haproxynode
    bind *:80
    mode http
    default_backend backendnodes
```

This configuration block specifies a frontend node named **haproxynode**, which is bound to all network interfaces on port 80(`bind *:80`). It will listen for HTTP connections(it is possible to use TCP mode for other purposes) and it will use the back-end **backendnodes**.

2. Add the back-end configuration:

```s
# /etc/haproxy/haproxy.cfg
backend backendnodes
    balance roundrobin
    option forwardfor
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option httpchk HEAD / HTTP/1.1\r\nHost:localhost
    server node1 192.168.1.3:8080 check
    server node2 192.168.1.4:8080 check
```

This defines **backendnodes** and specifies several configuration options:

- The `balance` setting specifies the *load-balancing strategy*

- The `forwardfor` option ensures the forwared request includes the actual client IP address

- The first `http-request` line allows the forwareded request to include the port of the client HTTP request. The second adds the proto-header containing https if `ssl_fc`, a HAProxy system variable, returns true. This will be the case if the connection was first made via an SSL/TLS transport layer

- `option httpchk` defines the check HAProxy uses to test if a web server is still valid for forwarding requests. If the server does not respond to the defined request it will not be used for load balancing until it passes the test

- The `server` lines define the actual server nodes and their IP addresses, to which IP addresses will be forwarded. The servers defined here are **node1** and **node2**, each of which will use the health check you have defined

3. Add the optional stats node to the configuration:

```s
# /etc/haproxy/haproxy.cfg

listen stats
    bind :32700
    stats enable
    stats uri /
    stats hide-version
    stats auth someuser:password
```

The HAProxy stats node will listen on port 32700(`bind :32700`) for connections and is configured to hide the version of HAProxy as well as to require a password login. Replace `password` with a more secure password. In addition, it is recommended to disable stats login in *production*

<br></br>

## Running and Monitoring

1. Restart the HAProxy service so that the new configuration can take effect:

```sh
sudo service hasproxy restart
```

Now, any incomfing requests to the HAProxy node at IP address will be forwarded to an internally networked node with pre-designated IP address. These backend nodes will serve the HTTP requests. If at any time either of these nodes fails the health check, they will not be used to serve any reqeusts *until they pass the test*.

In order to view statistics and monitor the health of the nodes, navigate to the IP address or domain name of the frontend node in a web browser at the assigned port, e.g., http://localhost:32700. This will display statistics such as the number of times a request was forwarded to a particular node as well as the number of current and previous sessions handled by the frontend node.