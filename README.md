# strongswan_ipsec_gre_rpi_ubuntu
strongswan_ipsec_gre_rpi_ubuntu

# ipsec_tunnel_gre
GRE (Generic Route Encapsulation) with IPsec.
Trouble with this one is (again): One peer is behind a NAT with a dynamically assigned IP address.
This is a typical setup for connecting a home network to a VPN server. So there are limitations:
- The tunnel may only be established by one side (peer that is located in the home network).


# Prerequisites
- strongswan installed on both machines
- According to https://wiki.strongswan.org/projects/strongswan/wiki/RouteBasedVPN
- You're root on the Ubuntu server (no need to sudo).
- Nice trick using a loopback interface on both sides for the IPsec connectionhttps://blog.vyos.io/setting-up-gre-slash-ipsec-behind-nat (because we need to figure out some traffic selectors).
- Closely related: https://wiki.strongswan.org/projects/strongswan/wiki/virtualip: If Virtual IPs (IP assigned to client by server) are used there *must not be any* traffic selectors.

# Setup
## RasPi
### IPsec part
- `sudo nano /etc/ipsec.conf`
```
conn <connname>
        right=<public ip of server to connect to>
        rightid=%<see above>
        # this one selects for gre traffic only   
        #rightsubnet=%dynamic[gre] 
        # this one will match all traffic
        rightsubnet=%dynamic
        # request address from peer/server
        # use ipv4
        leftsourceip=%config4
        # gre traffix only
        #leftsubnet=%dynamic[gre]
        leftsubnet=%dynamic
        auto=route 
        compress=no
        type=tunnel
        dpdaction=restart
        dpddelay=300s
        rekey=no
        keyexchange=ikev2
        ike=aes256-sha1-modp1024,3des-sha1-modp1024!
        esp=aes256-sha1,3des-sha1!
        fragmentation=yes
        forceencaps=yes
        authby=secret
```
- There has to be a matching secret PSK in `/etc/ipsec.secrets`

## Ubuntu
### IPsec part
- `nano /etc/ipsec.conf`
```
conn grepresharedkey
        dpdaction=clear
        dpddelay=30s
        rekey=no
        auto=route
        compress=no
        # transport is sufficient
        # we'll only encrypt the payload
        # since our ip header is not that interesting
        type=tunnel
        keyexchange=ikev2
        ike=aes256-sha1-modp1024,3des-sha1-modp1024!
        esp=aes256-sha1,3des-sha1!
        fragmentation=yes
        forceencaps=yes
        left=%any
        leftid=157.90.112.171
        # this ts selects for gre traffic only
        #leftsubnet=%dynamic[gre]
        # this one is for all traffic
        leftsubnet=dynamic
        right=%any
        rightid=%any
        # assign an ip to our peer connecting to us
        rightsourceip=10.8.5.0/24
        rightsubnet=%dynamic
```
- Don't forget about the matching shared secret in `/etc/ipsec.secrets`.

# Activation
## RPi
- `sudo systemctl restart strongswan`
- `sudo ipsec up <connname>
- `sudo ip tunnel add ipsec0 local 10.8.5.1 remote <remote server public ip> mode gre`
- `ip link set ipsec0 up`
- `ip route add <remote server public ip> dev ipsec0`

## Ubuntu
- `systemctl restart strongswan-starter`
- `ip tunnel add ipsec0 local <server public ip> remote 10.8.5.1 mode gre`
- `ip link set ipsec0 up`
- `ip route add 10.8.5.1 dev ipsec0`
-
# Verify
## RPi
- `sudo ipsec status`
- 

## Ubuntu
- `cat /var/log/syslog | grep charon`


# Alternate Setup (with an additional private address)
Server will have an IP Address of 10.8.9.1, Client (RasPi) will be 10.8.9.2
## RasPi
- create config:
```
conn <connName>
        right=<public ip of responder>
        rightid=%<public ip of responder>
        # this one selects for gre traffic only
        #rightsubnet=%dynamic[gre]
        rightsourceip=10.8.9.1
        # this needs to be dynamically assigned
        rightsubnet=%dynamic
        # we'd love ho have a certain ip
        # but there's no guarantee
        leftsourceip=10.8.9.2
        # gre traffix only
        #leftsubnet=%dynamic[gre]
        # traffic selector dynamic according to assigned ip
        leftsubnet=%dynamic
        auto=route
        compress=no
        type=tunnel
        dpdaction=restart
        dpddelay=60s
        rekey=no
        keyexchange=ikev2
        ike=aes256-sha1-modp1024,3des-sha1-modp1024!
        esp=aes256-sha1,3des-sha1!
        fragmentation=yes
        forceencaps=yes
        authby=secret
```
- create gre tunnel to between the virtual ip (our side) assigned by IPsec responder (public ip server) and the responder: `sudo ip tunnel add ipsec1 mode gre local 10.8.9.2 remote 157.90.112.171`
- bring it up (else no routing):  `sudo ip link set ipsec1 up`
- add route: `sudo ip route add 10.8.9.0/24 dev ipsec1`

## Ubuntu
- create config file:
```
conn grepresharedkey2
        dpdaction=clear
        dpddelay=30s
        rekey=no
        auto=route
        compress=no
        # transport is sufficient
        # we'll only encrypt the payload
        # since our ip header is not that interesting
        type=tunnel
        keyexchange=ikev2
        ike=aes256-sha1-modp1024,3des-sha1-modp1024!
        esp=aes256-sha1,3des-sha1!
        fragmentation=yes
        forceencaps=yes
        left=%any
        leftid=<our public ip>
        # this ts selects for gre traffic only
        #leftsubnet=%dynamic[gre]
        # festes subnet geht in die binsen
        leftsourceip=10.8.9.1
        leftsubnet=%dynamic
        right=%any
        rightid=%any
        # assign an ip to our peer connecting to us
        rightsourceip=10.8.9.2
        rightsubnet=%dynamic
        authby=secret
```
- We'll assign an additional address to our gre tunnel interface
- add tunnel interface: `ip tunnel add ipsec1 mode gre local <server public ip> remote 10.8.9.2`
- bring it up: `ip link set ipsec1 up`
- add route: `ip route add <whatever network is behind the tunnel on the client side, e.g. 192.168.3.0/24> dev ipsec1`
- add ip address to tunnel device so we may have adresses from the same subnet at both ends: `ip addr add 10.8.9.1 dev ipsec1`
