PyTorStemPrivoxy
================

Python, Tor, Stem, Privoxy program that requests new connections via Tor and thereby obtains new IP addresses as well.# Crawling Anonymously with Tor in Python #

*adapted from the article "[Crawling anonymously with Tor in Python](http://sacharya.com/crawling-anonymously-with-tor-in-python/)" by S. Acharya, Nov 2, 2013.*

The most common use-case is to be able to hide one's identity using TOR or being able to change identities programmatically, for example when you are crawling a website like Google and you donâ€™t want to be rate-limited or blocked via IP address.

## Tor ##

Install Tor.

```shell
sudo apt-get update
sudo apt-get install tor
sudo /etc/init.d/tor restart
```

*Notice that the socks listener is on port 9050.*

Next, do the following:

- Enable the ControlPort listener for Tor to listen on port 9051, as this is the port to which Tor will listen for any communication from applications talking to the Tor controller.
- Hash a new password that prevents random access to the port by outside agents.
- Implement cookie authentication as well.

You can create a hashed password out of your password using:
	
```shell
tor --hash-password my_password
```

Then, update the /etc/tor/torrc with the port, hashed password, and cookie authentication.

```shell
sudo gedit /etc/tor/torrc
```

```shell
ControlPort 9051
# hashed password below is obtained via `tor --hash-password my_password`
HashedControlPassword 16:E600ADC1B52C80BB6022A0E999A7734571A451EB6AE50FED489B72E3DF
CookieAuthentication 1
```

Restart Tor again to the configuration changes are applied.
	
```shell
sudo /etc/init.d/tor restart
```

## python-stem ##

Next, install `python-stem` which is a Python-based module used to interact with the Tor Controller, letting us send and receive commands to and from the Tor Control port programmatically.

```shell
sudo apt-get install python-stem
```

## privoxy ##

Tor itself is not a http proxy. So in order to get access to the Tor Network, use `privoxy` as an http-proxy though socks5.

Install `privoxy` via the following command:
	
```shell
sudo apt-get install privoxy
```

Now, tell `privoxy` to use TOR by routing all traffic through the SOCKS servers at localhost port 9050.

```shell
sudo gedit /etc/privoxy/config
```

and enable `forward-socks5` as follows:
	
```shell
forward-socks5 / localhost:9050
```

Restart `privoxy` after making the change to the configuration file.
	
```shell
sudo /etc/init.d/privoxy restart
```

##Python Script##

In the script below, `urllib2` is using the proxy. `privoxy` listens on port 8118 by default, and forwards the traffic to port 9050 upon which the Tor socks is listening.

Additionally, in the `renew_connection()` function,  a signal is being sent to the Tor controller to change the identity, so you get new identities without restarting Tor. Doing such comes in handy when crawling a web site and one doesnâ€™t wanted to be blocked based on IP address.

**[PyTorStemPrivoxy.py](https://gist.github.com/KhepryQuixote/46cf4f3b999d7f658853#file-pytorstemprivoxy-py)**

```python

'''
Python script to connect to Tor via Stem and Privoxy, requesting a new connection (hence a new IP as well) as desired.
'''

import stem
import stem.connection

import time
import urllib2

from stem import Signal
from stem.control import Controller

# initialize some HTTP headers
# for later usage in URL requests
user_agent = 'Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.9.0.7) Gecko/2009021910 Firefox/3.0.7'
headers={'User-Agent':user_agent}

# initialize some
# holding variables
oldIP = "0.0.0.0"
newIP = "0.0.0.0"

# how many IP addresses
# through which to iterate?
nbrOfIpAddresses = 3

# seconds between
# IP address checks
secondsBetweenChecks = 2

# request a URL 
def request(url):
    # communicate with TOR via a local proxy (privoxy)
    def _set_urlproxy():
        proxy_support = urllib2.ProxyHandler({"http" : "127.0.0.1:8118"})
        opener = urllib2.build_opener(proxy_support)
        urllib2.install_opener(opener)

    # request a URL
    # via the proxy
    _set_urlproxy()
    request=urllib2.Request(url, None, headers)
    return urllib2.urlopen(request).read()

# signal TOR for a new connection 
def renew_connection():
    with Controller.from_port(port = 9051) as controller:
        controller.authenticate(password = 'my_password')
        controller.signal(Signal.NEWNYM)
        controller.close()

# cycle through
# the specified number
# of IP addresses via TOR 
for i in range(0, nbrOfIpAddresses):

    # if it's the first pass
    if newIP == "0.0.0.0":
        # renew the TOR connection
        renew_connection()
        # obtain the "new" IP address
        newIP = request("http://icanhazip.com/")
    # otherwise
    else:
        # remember the
        # "new" IP address
        # as the "old" IP address
        oldIP = newIP
        # refresh the TOR connection
        renew_connection()
        # obtain the "new" IP address
        newIP = request("http://icanhazip.com/")

    # zero the 
    # elapsed seconds    
    seconds = 0

    # loop until the "new" IP address
    # is different than the "old" IP address,
    # as it may take the TOR network some
    # time to effect a different IP address
    while oldIP == newIP:
        # sleep this thread
        # for the specified duration
        time.sleep(secondsBetweenChecks)
        # track the elapsed seconds
        seconds += secondsBetweenChecks
        # obtain the current IP address
        newIP = request("http://icanhazip.com/")
        # signal that the program is still awaiting a different IP address
        print ("%d seconds elapsed awaiting a different IP address." % seconds)
    # output the
    # new IP address
    print ("")
    print ("newIP: %s" % newIP)

```

Execute the Python 2.7 script above via the following command:
	
```shell
python PyTorStemPrivoxy.py
```

When the above script is executed, one should see that the IP address is changing every few seconds.



## Adaptations to the original article ##

- *tweaks of grammar.*
- *the use of `python-stem` instead of `pytorctl`.*
- *a slight difference of settings within the `/etc/tor/torrc` file.*
- *the use of a different hashed password for the Tor controller, in this case `my_password`.*
- *some modifications in the sample program to accommodate the use of `python-stem`, cleaner logic, and more comprehensive commentary.*