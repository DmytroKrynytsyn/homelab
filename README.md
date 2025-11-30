# homelab
My homelab setup

<img width="1024" height="1536" alt="homelab" src="https://github.com/user-attachments/assets/81124133-b9ed-4ed0-b085-a2ae8a7b065b" />


Three metal PS, Wyse 5070, Debian, no desktop

## On all machines
/etc/hosts
dns of all home lab machines

## On client machine

ssh-keygen -t ed25519 -f ~/.ssh/homelab -C "homelab"
ssh-copy-id -i ~/.ssh/homelab.pub dmytro@remote_server_ip
ssh -i ~/.ssh/homelab dmytro@remote_server_ip

~/.ssh/config
Host myserver
    HostName remote_server_ip
    User dmytro
    IdentityFile ~/.ssh/homelab



## On each machine


### Network interfaces, static IPs
vi /etc/network/interfaces
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4


### DNS, only one
vi /etc/resolv.conf
nameserver 8.8.8.8

### Firewall, only explicit allowed

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

### Local sudo, just convinient
sudo visudo
dmytro ALL=(ALL) NOPASSWD:ALL



#Ansible

ansible all -i inventory.ini -b -a "uptime"
ansible-playbook -i inventory.ini basic.yml
