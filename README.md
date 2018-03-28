Intelie Challenge

In view of the proposed issue we have the following alternatives:

Iptables redirect

Since iptables's main purpose is to filter packets, we can use it on Server A to redirect the requests originating from Client A on port 80 and redirect them to Server B on port 8000.

We would use an explicit filter, ensuring that only packets coming from that source would be passed on.

Steps:

Check iptables default policy
iptables -L

The output should look something like this:

Chain INPUT (policy ACCEPT)
target		prot opt source	destination

Chain FORWARD (policy ACCEPT)
target		prot opt source	destination

Chain OUPUT (policy ACCEPT)
target		prot opt source	destination


As a good practice the best way to work with Iptables is to set the default policy as DROP and to release access to the required resources.

Let's change the default policy to drop

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP

Chain INPUT (policy ACCEPT)
target		prot opt source	destination

Chain FORWARD (policy ACCEPT)
target		prot opt source	destination

Chain OUPUT (policy ACCEPT)
target		prot opt source	destination

Now let's just release what they need that is ssh access, we will define a port other than the default (22) for security, port 5522 will be used.

let's edit the SSH service configuration file
vim / etc / ssh / sshd_config, edit the Port parameter as below:

Port 5522
ListemAddress 0.0.0.0

Restart the ssh service with the command below

systemctl restart sshd

Done this we will access the machine with ssh -p 5522 <host>

The next step is to enable packet redirection with the command below:

sysctl net.ipv4.ip_forward = 1

Let's free ports 5522 and 80 on Server A for connections originating from client A

iptables -I INPUT -p tcp - <client_A> --dport 5522 -j ACCEPT
iptables -I INPUT -p tcp - <A-client> --dport 80 -j ACCEPT

Now let's create the rules that will redirect the input and return of the incoming connections on port 80 originating from client A to port 8000 of Server B

iptables -t nat -A PREROUTING -d <A-server> -p tcp -dport 80 -j DNAT --to <B-server>: 80

iptables -t nat -A POSTROUTING -d <A-server> -p tcp -dport 80 -j SNAT --to <A-client>

Redirect via Mod Proxy Apache Web Server

Install an Apache Web server working with mod_proxy to forward it to port 80 of server A to port 8000 of server B

Let's install the apache service on Server A,

yum install httpd

We need to release access on port 80 for connections originating from Client A.

iptables -I INPUT p tcp -s <client_a> --dport 80 -j ACCEPT

Let's mount a Virtual Host file to configure the redirection.

vim /etc/httpd.conf/vhost.conf

<VirtualHost *:80>

    ProxyPreserveHost On
    ProxyPass 	    / http://<SERVIDOR_B:8000/
    ProxyPassReverse / http://<SERVIDOR_B>:8000/
    ServerName <SERVIDOR_A>.exemplo.com

</VirtualHost>

Save the file and run the command below to check if there are no errors in the configuration

Thus, all connections that come through a particular url and source will be forwarded to the application running on port 8000 of server B.

httpd -t

After that restart the apache service to apply the settings.

That way  when we connect from client A in the browser


Conclusion:

Both forms meet the situation, in the first we use iptables to filter the packets and forward iptables is very versatile for this purpose and allows to define. In this scenario, server A would act as a gateway, between client A and server B.

The other way would be to work with the Apache Web Server, widely used in the market is a feature prepared to stand facing the internet. Adding modules we can act with it in several ways, to solve this problem we use mod_proxy.

