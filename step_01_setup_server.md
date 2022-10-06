# Preparing the server

I bought a RS 2000 G9.5 a1 "Root"-Server which is a virtual machine with 6 dedicated CPU cores, 16 GB Ram and 320 GB of storage. It costs ~19€/month and should be good enough as my mail server.

The server was available wthin short time. I first reinstalled Debian 11 on it, using the "one large partition" scheme.

I also changed the reverse DNS name for the server IP to my MTA hostname `mail.mydomain.com` which is important later for delivering emails. And added an A-record fpor the MTA hostname. It is important that the MTS reverse and forward DNS lookups matches.


## Setting up the OS

For the following steps I usually use ansible. I will do the setup here manually.

### Set the hostname

`hostnamectl set-hostname MYHOSTNAME`

### Install pdns-recursor to be independent of the hosting provider's DNS servers.

```
apt -y install pdns-recursor
```

Edit `/etc/powerdns/recursor.conf` and enable/change the following lines:
```
# xxx.yyy.zzz.aaa - the IP of the server
allow-from=127.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10, 169.254.0.0/16, 192.168.0.0/16, 172.16.0.0/12, ::1/128, fc00::/7, fe80::/10, xxx.yyy.zzz.aaa/0
dnssec=validate
local-address=0.0.0.0
```

Now enable the recursor
```
systemctl disable --now systemd-resolved
systemctl enable --now pdns-recursor
rm /etc/resolv.conf # remove symlink to systemd resolve stuff
echo "nameserver 127.0.0.1" > /etc/resolv.conf
# test that DNS and DNSSEC works
dig @127.0.0.1 +adflag example.org
# The output should contain a line with 'flags: qr rd ra ad' - the 'ad' relevant
```

### Setup iptables and rules

```
apt -y install iptables iptables-persistent
```

`/etc/iptables/rules.v4`:
```
*filter
:INPUT DROP
:FORWARD ACCEPT
:OUTPUT ACCEPT

# Allow traffic of existing connections
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# Allow all traffic from localhost
-A INPUT -i lo -j ACCEPT
# Allow traffic within private networks
-A INPUT -s 10.0.0.0/8 -d 10.0.0.0/8 -j ACCEPT
-A INPUT -s 172.16.0.0/12 -d 172.16.0.0/12 -j ACCEPT
-A INPUT -s 192.168.0.0/16 -d 192.168.0.0/16 -j ACCEPT
# Allow ICMP
-A INPUT -p icmp -j ACCEPT
# Allow TCP ping (Echo)
-A INPUT -p tcp --dport 7 -j ACCEPT

# Allow SSH
-A INPUT -p tcp --dport 22 -j ACCEPT
# Allow kubernetes apiserver
-A INPUT -p tcp --dport 6443 -j ACCEPT

COMMIT
```

`/etc/iptables/rules.v6`:
```
*filter
:INPUT DROP
:FORWARD ACCEPT
:OUTPUT ACCEPT

# Cleanup
-F INPUT
# Allow traffic of existing connections
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# Allow all traffic from localhost
-A INPUT -i lo -j ACCEPT
# Allow ICMP
-A INPUT -p icmpv6 -j ACCEPT

COMMIT
```

```
systemctl restart netfilter-persistent
```

### Load the br_netfilter

This is required for kubernetes networking

```
modprobe br_netfilter
echo br_netfilter >> /etc/modules
```

## Install K3S

The k3s binary can be downloaded by the installer script:

```
apt -y install curl
curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_ENABLE=true sh -
```

I prefer having all relevant k3s files in /data, so I create it symlink to it:

```
mkdir -p /etc/rancher/k3s
mkdir -p /data/k3s
touch /data/k3s/config.yaml
ln -s /data/k3s/config.yaml /etc/rancher/k3s/config.yaml
```

Create a resolv.conf file for coredns at `/data/k3s/resolv.conf` where `xxx.yyy.zzz.aaa` is the IP of your server. Don't use 127.0.0.1 since this would cause a loop in coredns.

```
nameserver xxx.yyy.zzz.aaa
```

Configure k3s by editing `/etc/rancher/k3s/config.yaml`:

```yaml
data-dir: /data/k3s
write-kubeconfig: /data/k3s/kubeconfig
default-local-storage-path: /data/volumes
disable-helm-controller: true
disable:
  - traefik
resolv-conf: /data/k3s/resolv.conf
tls-san:
  - "mail.mydomain.com"
```

Now you can start k3s and follow it's logs until it's fully initialized:

```
systemctl enable --now k3s
journalctl -f -u k3s
```

## Verify the installation

Deploy a test container and check that it works and has internet access.

```
# note that the image must be one that runs a command and does not exit immediately. nginx will do it
kubectl create deployment test --image=nginx
kubectl get pods
kubectl exec test-764c85dd84-klbcd -- sh -c "curl -s ipconfig.io"
kubectl delete deployment test
```

