# bridge

brigde is a dynamic port forwarder over HTTP(S) (with HTTP PROXY support)

In some places, network is locked behind a firewall and Internet connection is available only by using a proxy server. If you wishes to connect to your SSH at home, you're in trouble. However, there is a simple solution to this: [tunneling over HTTPS](http://www.google.com.br/search?q=ssh+over+http+proxy). However, if you are one of those lucky guys that cannot use tunneling over HTTPS, this page can help you.

Using any protocol that can exchange information, it is possible to encapsulate a connection over it. Some not-so-common examples are: [IP-over-DNS](http://thomer.com/howtos/nstx.html), [IP-over-ICMP](http://thomer.com/icmptx/). This page shows a TCP tunneling solution (like ssh -L option) (ab)using HTTP.

The program is divided in two parts: the first one implements a HTTP server, that can be setup to run on any server. However, it is generally easier to have ports 80 or 8080 as authorized ports in your proxy server.
The seconds part is the client program. It opens a local TCP port or reads STDIN. After a connection is received, it connects to the server program just like a browser would do and exchange packages using HTTP(S) requests (in this case: GET, PUT, POST, DELETE). 
How to run

Download bridge file (attachments bellow this page). It is a ruby script that implements both server and client. It depends on webrick and net:http that are also ruby on rails dependencies. Run it as any other ruby script. Server is activated when two parameters are passed: local server port and relative URL

```
bridge$ ruby bridge 8080 /bridge
```

This brings up Webrick running in port 8080 and answering bridge requests at http://myserver:8080/bridge
If you wish to use SSL it is recommended to set up SSL-offload proxy server in front of the bridge. Bridge server itself does not support SSL.
Now, it is time to run the client program:

```
client$ ruby bridge 8022 http://myserver:8080/bridge mysshserver.xxx.com 22
or
client$ ruby bridge 8022 https://myserver/bridge mysshserver.xxx.com 22
```

If everything goes OK, the client program starts to listen on port 8022 (user "-" intead of 8022 for STDIN/STDOUT). When someone connects to it, it translates the communication with HTTP commands and forwards them to myserver that effectively connects to mysshserver.xxx.com at port 22. As 22 is a ssh port, one can use this tunneling using:

```
client$ ssh localhost -p 8022
Password:

mysshserver $
```

Now, someone might like to use SSH to setup a SOCKS server, a reverse TCP port, etc or just connect to a remote VPN server.

This command can also be used as ProxyCommand in OpenSSH client. In this casem the use of bridge is more "transparent".

```
~/.ssh/config
Host *.xxx.com
    ProxyCommand bridge - http://myserver:8080/bridge %h %p
Host mysshserver2
    ProxyCommand bridge - http://myserver:8080/bridge mysshserver2.noip.org 8022
```

And call ssh as usual. For any xxx.com server:

```
client$ ssh mysshserver.xxx.com
Password:

mysshserver $
```

And for server2:

```
client$ ssh mysshserver2
Password:

mysshserver2 $
```

If you need that client connect to bridge over a HTTP Proxy, you can set the standard environment vars like http_proxy.

```
client$ http_proxy=http://myuser:pass@myproxy:3128 ruby bridge 8022 http://myserver:8080/bridge mysshserver.xxx.com 22
```
