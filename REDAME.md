# Mastering Tor

---

**WARNING**\
These are drafts. One day I'll arrange them. In the meanwhile, if you want to help me or if you have some tips to harden the Tor configuration, please tell me! I need to know people like you.

---

# What is it?
The Tor network is a group of volunteer-operated servers that allows people to improve their privacy and security on the Internet.

# Onion services
Tor's onion services let users publish web sites and other services without needing to reveal the location of the site.

# Why Tor?
Internet data packets have two parts: a data payload and a header used for routing. The data payload is whatever is being sent, whether that's an email message, a web page, or an audio file. Even if you encrypt the data payload of your communications, traffic analysis still reveals a great deal about what you're doing and, possibly, what you're saying. That's because it focuses on the header, which discloses source, destination, size, timing, and so on. 
The aim of Tor is to reduce traffic analysis, making the header meno ricco di informazione.
Nicolò, consider your actual situation: SSH is configured to perfectly manage the payload of internet packets. You'll configure Tor in order to manage the header of those packets. Wow!

# [Install Tor on Debian and derivates](https://2019.www.torproject.org/docs/debian.html.en#ubuntu)
1. apt-transport-tor: to use source lines with https:// in `/etc/apt/sources.list` the apt-transport-https package is required. Install it with:
```
sudo apt install apt-transport-https
```
2. sources.list: You'll need to set up our package repository before you can fetch Tor. You need to add the following entries to `/etc/apt/sources.list` or a new file in `/etc/apt/sources.list.d/`.
**WARNING**: first, you have to find out the name of your OS distribution. You can get it executing `lsb_release -cs`. Replace 'buster' with your distribution name in the following two commands, or you'll get errors later!
```
deb https://deb.torproject.org/torproject.org buster main
deb-src https://deb.torproject.org/torproject.org buster main
```
3. Add the gpg key used to sign the packages by running the following commands at your command prompt:
```
sudo su
curl https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | gpg --import
gpg --export A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89 | apt-key add -
```
4. We provide a Debian package to help you keep our signing key current. It is recommended you use it. Install it with the following commands:
```
sudo apt update
sudo apt install tor deb.torproject.org-keyring
```
5. Now Tor is installed and running. Check the installed version with `tor --version`.

# [Install an onion service](https://medium.com/@tzhenghao/how-to-ssh-over-tor-onion-service-c6d06194147)
Assume that NAME is the name of the hidden service you want to create. For example, configuring ssh as an hidden service, I decided to simply call it 'ssh'.
1. Create a directory for the new hidden service: `sudo mkdir /var/lib/tor/NAME`
2. You have to ensure that all the files and folders into that tor directory are owned by the debian-tor user. Do that with chown: `sudo chown -R debian-tor:debian-tor /var/lib/tor`. The '-R' flag says to change the ownership recursively. I don't know if really needed but I also executed `sudo chmod 700 -R /var/lib/tor `. Check that all went good with `sudo ls -l /var/lib/tor`, you should get something like this for the NAME folder:
`drwxr-sr-x 2 debian-tor debian-tor    4096 Feb 16 17:00 ssh`
3. Modify Tor config file `sudo nano /etc/tor/torrc`, adding the following lines:
```
HiddenServiceDir /var/lib/tor/NAME/
HiddenServiceVersion 3
HiddenServicePort 22 127.0.0.1:22
# HiddenServiceAuthorizeClient stealth NAME
```
HiddenServiceDir basically tells tor that you have/want a hidden service directory with the proper configs based on the given path.

---

HiddenServiceVersion
Since Tor 0.3.2 and Tor Browser 7.5.a5 56-character long v3 onion addresses are supported and should be used instead. This newer version of onion services ("v3") features many improvements over the legacy system:

    Better crypto (replaced SHA1/DH/RSA1024 with SHA3/ed25519/curve25519)
    Improved directory protocol, leaking much less information to directory servers.
    Improved directory protocol, with smaller surface for targeted attacks.
    Better onion address security against impersonation.
    More extensible introduction/rendezvous protocol.
    A cleaner and more modular codebase.

For details see [Why are v3 onions better](https://trac.torproject.org/projects/tor/wiki/doc/HiddenServiceNames). You can identify a next-generation onion address by its length: they are 56 characters long, as in 4acth47i6kxnvkewtm6q7ib2s3ufpo5sqbsnzjpbi7utijcltosqemad.onion. The specification for next gen onion services can be found here. 

---

HiddenServicePort here should be port 22, since that’s the default port for ssh. You can change this to any other value.
HiddenServiceAuthorizeClient basically tells tor to authorize a client that wants to make a connection to the specified hidden service. The stealth command basically tells tor that you want this node to be hidden from all othertor nodes in the network.
What these lines mean!?
WARNING! The line `# HiddenServiceAuthorizeClient stealth NAME` is not compatible with version 3 of the hidden service protocol, so you have to comment it.
4. Restart Tor: `sudo /etc/init.d/tor restart`
5. Navigate to the hidden service directory /var/lib/tor/NAME again, and you should see that tor has populated the directory with 3 files: client\_keys, hostname and private\_key. If they are not here, something went wrong!
6. Save somewhere the onion address stored in /var/lib/tor/NAME/hostname. I'll call it ONION_ADDRESS
7. Open a client (with Tor installed). Then, execute: `torsocks ssh USERNAME@ONION_ADDRESS` where USERNAME is the username of the user to which you want to log in using SSH.
8. I don't know how, but it works! I tested it connecting my client to the Vodafone network and leaving the Raspberry linked to the modem. Crazy happiness!

---

**Tip**: when you'll want to connect to your onion service using torsocks, it is possible that you have to restart the tor service on your client: `sudo /etc/init.d/tor restart`

---

## Creating private V3 onion services

In order to make sure that only the clients you want can try to connect to the service (it isn't really necessary because the attacker should know the onion address and also have my ssh private key ahahah impossible) you should follow this guides:
- [Tor official](https://2019.www.torproject.org/docs/tor-onion-service#ClientAuthorization)
- [A geek guide](https://matt.traudt.xyz/p/FgbdRTFr.html)

I'm following the geek guide. First of all, you should have already created the V3 onion service: this part of the guide will teach us how to make that service private.
We have to generate an x25519 key pair. If we want to use the Pyhton script provided by the geek, we have to:
- install python3 (probably already installed): `sudo apt install python3`
- install pip-python3: `sudo apt install pip-python3`
- install PyNaCl (it is the library that the geek's script uses in order to generate the key pair): `pip3 install pynacl`

Now, we have to create a python script that contains the code of the geek (maybe we can directly download that script but I don't know how to do that... very sad...).
Suppose our script is called 'keygen.py'. Execute it with `python3 keygen.py` and we'll get our brand-new key pair! Store it in a secure place.

Go back and continue reading the geek's guide.
Always ensure that the owner of the files into the tor directory is debian-tor! For example, when you create the new file into authorized\_clients, probably it will be owned by root. No buono! Change it: `sudo chown -R debian-tor:debian-tor /var/lib/tor/ssh/authorized_clients`

I followed the guide: now I have authentication at Tor-level for my onion service. Wow!I checked that all is okay: I tried to remove the private key from the client and connect to the onion service: it refuses the connection and it is exactly what I want.

# Sources
[Tor overview](https://2019.www.torproject.org/about/overview.html.en)
[Onion service protocol](https://2019.www.torproject.org/docs/onion-services)
[Configure onion services](https://2019.www.torproject.org/docs/tor-onion-service.html.en)
[Running a Tor relay](https://2019.www.torproject.org/docs/tor-doc-relay.html.en)
[Onion services best practices](https://riseup.net/en/security/network-security/tor/onionservices-best-practices)
[OnionScan](https://github.com/s-rah/onionscan)
[The Vanguards onion service addon](https://riseup.net/en/security/network-security/tor/onionservices-best-practices)