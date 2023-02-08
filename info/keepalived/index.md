# ViReR.NeT Keepalived: A very simple IP failover setup

Here is a very quick documentation to configure a simple IP failover between two hosts.<br>

<br>
### Preliminary notes:
The network interface are named "eth0" if it differ on your host no worries, just adap the configuration as you need.<br>
- Our network have the netmask 255.255.255.0 (or a /24 prefix if you like)<br>
- Our primary node have the IP 192.168.0.2<br>
- Our secondary node have the IP 192.168.0.3<br>
- Our virual IP address (or VIP) will be 192.168.0.1<br>

<br>

First open the firewall to allow the VRRP protocol:
```bash
firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
firewall-cmd --reload
```

<br>

Here is the required packages for both node (as always the package name may change on your distro):<br>

```bash
# dnf install keepalived
```

And for debian like:
```bash
# apt-get install keepalived
```

### Primary node configuration:

edit the file located here **/etc/keepalived/keepalived.conf** and remove all the default content(important) and put the following content inside:

```bash
! Configuration File for keepalived

vrrp_instance VI_1 {
        state MASTER
        interface eth0
        virtual_router_id 51
        priority 255
        advert_int 1
        authentication {
              auth_type PASS
              auth_pass MySuper_Passw0rd
        }
        virtual_ipaddress {
              192.168.0.1/24
        }
}
```

<br>

Then go on the secondary node, 

edit and remove all default content(important) inside the file located here /etc/keepalived/keepalived.conf and put the follwowing lines:

```bash
! Configuration File for keepalived

vrrp_instance VI_1 {

        state BACKUP
        interface eth0
        virtual_router_id 51
        priority 254
        advert_int 1
        authentication {
              auth_type PASS
              auth_pass MySuper_Passw0rd
        }
        virtual_ipaddress {
              192.168.0.1/24
        }
}
```

<br>

Once configured, start keepalived service on secondary node.

#### Note: 
A good way to test, is to start the secondary/backup node before the primary as the VIP will move to the primary when he start

```bash
# systemctl start keepalived
```

<br>

Verify the network configuration on the secondary node:

```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:00:00:02:22:22 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.3/24 brd 192.168.0.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 192.168.0.1/24 scope global eth0
       valid_lft forever preferred_lft forever
```

<br>

Now, start keepalived service on primary node.

```bash
# systemctl start keepalived
```

Verify the network configuration on the primary node:

```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:00:00:01:23:45 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.2/24 brd 192.168.0.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 192.168.0.1/24 scope global eth0
       valid_lft forever preferred_lft forever

```

<br>

Now, go back on secondary node and you should not see the VIP active on that node:<br>

```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:00:00:02:22:22 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.3/24 brd 192.168.0.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
```

<br>

### Last words:
- If you need to enable multiple VIP inside the same LAN you MUST change the "virtual_router_id" parameter to avoid conflict.
- There is many other configuration options, for example to start script or to notify people when failover event occured, so checkout the documenations, on this page it is a very basic setup.

