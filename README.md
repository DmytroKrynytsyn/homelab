# homelab
My homelab setup

Three metal PS, Wyse 5070, Debian, no desktop

## On all machines
/etc/hosts
dns of all home lab machines

## On client machine
~~~
ssh-keygen -t ed25519 -f ~/.ssh/homelab -C "homelab"
ssh-copy-id -i ~/.ssh/homelab.pub dmytro@remote_server_ip
ssh -i ~/.ssh/homelab dmytro@remote_server_ip

~/.ssh/config
Host myserver
    HostName remote_server_ip
    User dmytro
    IdentityFile ~/.ssh/homelab
~~~


## On each machine


### Network interfaces, static IPs


### /etc/network$ sudo cat interfaces
~~~
/etc/network# cat interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
#allow-hotplug wlp0s12f0
#iface wlp0s12f0 inet static
#	address 192.168.178.92
#	netmask 255.255.255.0
#	gateway 192.168.178.1
#	dns-nameservers 8.8.8.8 8.8.4.4
#	wpa-ssid <name>
#	wpa-psk <key>

auto enp1s0
iface enp1s0 inet static
	address 192.168.178.102
	inetmask 255.255.255.0
	gateway 192.168.178.1
	dns-nameservers 8.8.8.8 8.8.4.4

~~~



### DNS, only one
~~~
vi /etc/resolv.conf
nameserver 8.8.8.8
~~~

### Firewall, only explicit allowed
~~~
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22
sudo ufw enable
sudo ufw status

### Harden SSH
/etc/ssh/sshd_config

PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
PubkeyAuthentication yes
PermitRootLogin no
~~~

### Local sudo, just convinient
~~~
sudo visudo
dmytro ALL=(ALL) NOPASSWD:ALL
~~~


# Ansible   
~~~
ansible all -i inventory.ini -b -a "uptime"
ansible-playbook -i inventory.ini basic.yml
~~~