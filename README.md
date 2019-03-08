# Resolving Route 53 private hosted zones from an on-premises network

## Setting up DNS forwarder on the AWS

To resolve AWS Private Hosted Zones on our on-premises network we need to setup DNS forwarder on our VPC. Before that you need to have VGW (Virtual Private Gateway) set between your VPC and on-premises network.

1. We need to enable DNS resolution and DNS hostnames on our VPC.

2. We need to setup a EC2 instance on the VPC, best with Ubuntu 18.04 operating system. For this case we can use t2.micro instance.

3. After the EC2 instance is up and running we should SSH into the instance and issue `sudo apt-get update` command.

4. Next install BIND with `sudo apt-get install bind9 bind9utils bind9-doc`.

5. Open `/etc/bind/named.conf.options` with the editor of your choice and paste next configuration (notice the 10.0.0.0/8 - this should be the list of addresses you trust).

```
acl "trusted" {
    10.0.0.0/8; # <-- edit according to your on-premises setup
    localhost;
    localnets;
};

options {
        directory "/var/cache/bind";
        recursion yes;
        allow-query { trusted; };

        forwarders {
                10.12.0.2; # <-- edit according to your VPC setup, if your VPC CIDR is 192.168.0.0/16, then this should be 192.168.0.2
        };

        forward only;

        dnssec-enable no;
        dnssec-validation no;
        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };
};
```

6. Test the BIND configuration with `sudo named-checkconf` and restart service with `sudo service bind9 restart` for changes to take effect.

## For OSX users:

1. Update your brew `brew update`

2. Install dnsmasq `brew install dnsmasq`

3. In `/usr/local/etc/dnsmasq.conf` find a line `#conf-dir=/usr/local/etc/dnsmasq.d,*.conf` and uncomment it

4. Create directory `mkdir -p /usr/local/etc/dnsmasq.d`

5. Within that directory create a file with `touch /usr/local/etc/dnsmasq.d/development.conf`

6. Open file with `sudo nano /usr/local/etc/dnsmasq.d/development.conf`

7. Paste into the file `server=/example.internal/10.12.0.176` (this is Private IP of the DNS forwarder EC2 instance, that you have setup in the first section)

8. After that create folder as `sudo mkdir /etc/resolver` and create empty file with `touch /etc/resolver/internal`

9. Open file with `sudo nano /etc/resolver/internal` and paste `nameserver 127.0.0.1` into the file

10. Restart the dnsmasq service with `sudo brew services restart dnsmasq`

11. Wait a while and test your configuration with `dns-sd -G v4 myservice.example.internal`. You should see something like

```
DATE: ---Fri 08 Mar 2019---
 8:24:24.169  ...STARTING...
Timestamp     A/R Flags if Hostname                               Address                                      TTL
 8:24:24.200  Add     2  0 myservice.example.internal. 10.12.2.120                                  77
```
