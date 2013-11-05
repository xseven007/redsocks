REDSOCKS
=========
This is a modified version of original redsocks.
This variant is useful for anti-GFW (Great Fire Wall).
This is a modified version of original PREROUTING.
This is a modified version which is work on OpenWrt or Tomato and DD-WRT.
It provides the following advanced features:

Running
=======

Program has following command-line options:
 -c   sets proper path to config file ("./PREROUTING.conf" is default one)
 -t   tests config file syntax
 -p   set a file to write the getpid() into

Following signals are understood:
SIGUSR1 dumps list of connected clients to log
SIGTERM and SIGINT terminates daemon, all active connections are closed

You can see configuration file example in PREROUTING.conf.example
##Note:
Method 'autosocks5' and 'autohttp-connect' are removed.
To use the autoproxy feature, please change the redsocks section in
configuration file like this:

	redsocks {
	 local_ip = 192.168.1.1;
	 local_port = 1081;
	 ip = 192.168.1.1;
	 port = 9050;
	 type = socks5; // I use socks5 proxy for GFW'ed IP
	 autoproxy = 1; // I want autoproxy feature enabled on this section.
	                // The two lines above have same effect as
	                //    type = autosocks5;
	                // in previous release.
	 // timeout is meaningful when 'autoproxy' is non-zero.
	 // It specified timeout value when trying to connect to destination
	 // directly. Default is 10 seconds. When it is set to 0, default
	 // timeout value will be used.
	 timeout = 10;
	 //type = http-connect;
	 //login = username;
	 //password = passwd;
	}

##Work with GoAgent
To make redsocks2 works with GoAgent proxy, you need to set proxy type as
'http-relay' for HTTP protocol and 'http-connect' for HTTPS protocol  
respectively.
Suppose your goagent local proxy is running at the same server as redsocks2,
The configuration for forwarding connections to GoAgent is like below:

	redsocks {
	 local_ip = 192.168.1.1;
	 local_port = 1081; //HTTP should be redirect to this port.
	 ip = 192.168.1.1;
	 port = 8080;
	 type = http-relay; // Must be 'htt-relay' for HTTP traffic. 
	 autoproxy = 1; // I want autoproxy feature enabled on this section.
	 // timeout is meaningful when 'autoproxy' is non-zero.
	 // It specified timeout value when trying to connect to destination
	 // directly. Default is 10 seconds. When it is set to 0, default
	 // timeout value will be used.
	 timeout = 10;
	}
	redsocks {
	 local_ip = 192.168.1.1;
	 local_port = 1082; // HTTPS should be redirect to this port.
	 ip = 192.168.1.1;
	 port = 8080;
	 type = http-connect; // Must be 'htt-connect' for HTTPS traffic. 
	 autoproxy = 1; // I want autoproxy feature enabled on this section.
	 // timeout is meaningful when 'autoproxy' is non-zero.
	 // It specified timeout value when trying to connect to destination
	 // directly. Default is 10 seconds. When it is set to 0, default
	 // timeout value will be used.
	 timeout = 10;
	}

iptables example
================

# Create new chain
iptables -t nat -N PREROUTING

# Ignore your PREROUTING server's addresses
# It's very IMPORTANT, just be careful.
iptables -t nat -A PREROUTING -d 123.123.123.123 -j RETURN

# Ignore LANs and any other addresses you'd like to bypass the proxy
# See Wikipedia and RFC5735 for full list of reserved networks.
# See ashi009/bestroutetb for a highly optimized CHN route list.
iptables -t nat -A PREROUTING -d 0.0.0.0/8 -j RETURN
iptables -t nat -A PREROUTING -d 10.0.0.0/8 -j RETURN
iptables -t nat -A PREROUTING -d 127.0.0.0/8 -j RETURN
iptables -t nat -A PREROUTING -d 169.254.0.0/16 -j RETURN
iptables -t nat -A PREROUTING -d 172.16.0.0/12 -j RETURN
iptables -t nat -A PREROUTING -d 192.168.0.0/16 -j RETURN
iptables -t nat -A PREROUTING -d 224.0.0.0/4 -j RETURN
iptables -t nat -A PREROUTING -d 240.0.0.0/4 -j RETURN

# Anything else should be redirected to PREROUTING's local port
iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-ports 12345

# Apply the rules
iptables -t nat -A OUTPUT -p tcp -j PREROUTING

# Any tcp connection made by `luser' should be redirected.
root# iptables -t nat -A OUTPUT -p tcp -m owner --uid-owner luser -j PREROUTING

# You can also control that in more precise way using `gid-owner` from
# iptables.
root# groupadd socksified
root# usermod --append --groups socksified luser
root# iptables -t nat -A OUTPUT -p tcp -m owner --gid-owner socksified -j PREROUTING

# Now you can launch your specific application with GID `socksified` and it
# will be... socksified. See following commands (numbers may vary).
# Note: you may have to relogin to apply `usermod` changes.
luser$ id
uid=1000(luser) gid=1000(luser) groups=1000(luser),1001(socksified)
luser$ sg socksified -c id
uid=1000(luser) gid=1001(socksified) groups=1000(luser),1001(socksified)
luser$ sg socksified -c "firefox"

# If you want to configure socksifying router, you should look at
# doc/iptables-packet-flow.png and doc/iptables-packet-flow-ng.png
# Note, you should have proper `local_ip' value to get external packets with
# PREROUTING, default 127.0.0.1 will not go. See iptables(8) manpage regarding
# REDIRECT target for details.
# Depending on your network configuration iptables conf. may be as easy as:
root# iptables -t nat -A PREROUTING --in-interface eth_int -p tcp -j PREROUTING
