# Kubernetes in CentOS 7.5 on VirtualBox
A workflow for setting up a simple Kubernetes cluster using 3 VMs, a master and 2 workers. 

## Setup

The first section assumes you have VirtualBox installed and a CentOS 7.5 Minimal iso on hand.

1. Create Host Only Adapter (IP: 192.168.56.1/24 & No DHCP)
2. Create 1 Master VM
    - VirtualBox Settings
        - 1st network adapter set to Host Only
        - 2nd network adapter set to NAT
    - CentOS install settings:
        - Set timezone
        - Networking disabled and unconfigured
        - No users besides root
3. Create 2 Worker VMs
    - VirtualBox Settings
        - 1st network adapter set to Host Only
    - CentOS install settings:
        - Set timezone
        - Networking disabled and unconfigured
        - No users besides root
4. Configure networking
    - Master
        - `nmcli con add type ethernet con-name wan-con ifname enp0s8`
        - `nmcli con add type ethernet con-name lan-con ifname enp0s3 ip4 192.168.56.101/24`
        - Option 1: IPTables
            - `systemctl stop firewalld && systemctl disable firewalld`
            - `yum install -y iptables-services`
            - `systemctl enable iptables.service`
            - `echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf`
            - `sysctl -p /etc/sysctl.conf`
            - `iptables -A FORWARD -i enp0s3 -j ACCEPT`
            - `iptables -A FORWARD -o enp0s3 -j ACCEPT`
            - `iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE`
            - `service iptables save`
        - Option 2: Firewalld
            - `nmcli con mod lan-con connection.zone internal` 
            - `nmcli con mod wan-con connection.zone external`
            - `nmcli con up lan-con`
            - `nmcli con up wan-con`
    - Worker 1
        - `nmcli con add type ethernet con-name lan-con ifname enp0s3 ip4 192.168.56.102/24 gw4 192.168.56.101 ipv4.dns 8.8.8.8`
        - `nmcli con up lan-con`
    - Worker 2
        - `nmcli con add type ethernet con-name lan-con ifname enp0s3 ip4 192.168.56.103/24 gw4 192.168.56.101 ipv4.dns 8.8.8.8`
        - `nmcli con up lan-con`
5. Set hostnames
    - Master: `hostnamectl set-hostname kmaster`
    - Worker 1: `hostnamectl set-hostname kworker1`
    - Worker 2: `hostnamectl set-hostname kworker2` 
6. Create "centos" user on master
    - `adduser centos`
    - `passwd centos`
    - `usermod -aG wheel centos`
7. Configure keys for SSH
    - on local machine: 
        - `ssh-copy-id root@192.168.56.101`
        - `ssh-copy-id centos@192.168.56.101`
        - `ssh-copy-id root@192.168.56.102`
        - `ssh-copy-id root@192.168.56.103`

## Ansible
This section uses the provided Ansible YML files to install Kubernetes on the 3 nodes. 

1. Change values in hosts file
2. `ansible-playbook -i hosts kube-dependencies.yml`
3. `ansible-playbook -i hosts master.yml`
4. Check master node is ready
    - `ssh centos@master_ip`
    - `kubectl get nodes`
5. `ansible-playbook -i hosts workers.yml`
6. Check workers have joined
    - `ssh centos@master_ip`
    - `kubectl get nodes`

## References
Much of the ansible yml was used from the following:
- https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-1-10-cluster-using-kubeadm-on-centos-7