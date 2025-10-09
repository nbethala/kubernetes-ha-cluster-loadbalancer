## Setup Load Balancer on EC2 Instance : HAproxy setup instructions 

Login to the loadbalancer node
Switch as root -  sudo -i
Update your repository and your system
sudo apt-get update && sudo apt-get upgrade -y
Install haproxy
sudo apt-get install haproxy -y
Edit haproxy configuration
sudo vi /etc/haproxy/haproxy.cfg
make sure to use private ips for all the communication as we are using as we are going to perform all the communication within the vpc only

frontend kubernetes-api
         bind *:6443
         mode tcp
         option tcplog
         default_backend kube-master-nodes
What this means:
frontend kubernetes-api

This names the frontend block kubernetes-api. It's just an identifier — you could call it anything, but naming it clearly helps for readability.

bind *:6443

This tells HAProxy to listen for incoming connections on port 6443 on all interfaces (* means all available IPs on the server).

Port 6443 is the standard Kubernetes API port.
This is the port kubeadm and kubectl will connect to.
default_backend kube-masters

Any connection received on this frontend will be forwarded to the kube-masters backend (defined in the next block).

💡 This block is essentially the public-facing listener for Kubernetes API requests

backend kube-master-nodes
        mode tcp
        balance roundrobin
        option tcp-check
        default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100

    
        server master1 <private-ip of master node 01>:6443 check
        server master2 <private-ip of master node 02>:6443 check
What this means:
backend kube-masters

This names the backend pool kube-masters. This matches the default_backend in the frontend section.

mode tcp

HAProxy operates in TCP mode (not HTTP). The Kubernetes API server uses raw TCP (HTTPS), so this mode is required.

balance roundrobin

Load balances incoming connections using round-robin strategy:

First request → master1
Second request → master2
Third → master3
Then it repeats...
This spreads the load evenly across your control plane nodes.

option tcp-check

Enables health checks using TCP connection attempts. If HAProxy cannot make a TCP connection to a node’s port 6443, it considers that node unhealthy and removes it from the rotation.

default-server inter 5s fall 3 rise 2

These are health check tuning parameters:

inter 10s: Check every 10s.
downinter 5s: If down, recheck every 5s.
rise 2: Need 2 successful checks to be considered healthy.
fall 2: 2 failed checks mark the server as down.
slowstart 60s: After recovery, slowly ramp up traffic over 60s.
maxconn 250: Max 250 connections.
maxqueue 256: Queue limit.
weight 100: Default weight for load balancing.
final output

global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
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

frontend kubernetes-api
         bind *:6443
         mode tcp
         option tcplog
         default_backend kube-master-nodes

backend kube-master-nodes
        mode tcp
        balance roundrobin
        option tcp-check
        default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100

    
        server master1 <private-ip of master node 01>:6443 check
        server master2 <private-ip of master node 02>:6443 check
make sure to replace the placeholders with private ip of master instances

After Configuration
Save and Restart HAProxy:

sudo systemctl restart haproxy
Enable at boot:

sudo systemctl enable haproxy
Check if the port 6443 is open and responding for haproxy connection or not

nc -v localhost 6443
you will see a response like

Connection to localhost (127.0.0.1) 6443 port [tcp/*] succeeded!`
