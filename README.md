# CyberEscape-DoS-attack
CyberEscape Approach to Advancing Hard and Soft Skills in Cybersecurity Education
# 1. Overview
The virtual laboratory is a central component of the CyberEscape game. Virtualization technologies enabled the development of a controlled environment to simulate DoS attacks safely. Moreover, this environment could be easily scaled according to the number of students. The CyberEscape utilized bare metal virtualization and nested virtualization technologies. Bare metal virtualization uses the open-source Proxmox Virtual Environment (Proxmox VE) as a hypervisor based on the KVM hypervisor and Linux containers (LXC). <br>
Proxmox VE supports all the infrastructure necessary for DoS simulation, e.g. virtual machines, containers, virtual networks, network rate limit, and centralized management of DoS scripts. The DoS attack was performed against the fictional company environment developed using nested virtualization. Each student group had an Ubuntu Desktop 22.04 virtual machine with the Apache Web server deployed in a dedicated nested Proxmox VE hypervisor. Any remote communication with the virtual machine was lost during a DoS attack, and services like VNC, and SSH were unavailable. Therefore, nested virtualization enabled direct connection to the virtual machine from the Proxmox VE hypervisor console. <br> The common architecture can be seen in Figure 1: <br><br>
![image](https://github.com/MartinsRTU/CyberEscape-DoS-/assets/169056170/6fdc89f7-1682-4ae4-826e-72888ef2163e)
<br><br>
Figure 1. Common hypervisor and virtual machine architecture.
<br><br>
CyberEscape participants were able to trace and analyze this DoS attack using the network protocol analyzer Wireshark. They were expected to block the attack using a virtual machine hypervisor firewall that simulates the company's main firewall in real life. DoS scripts were executed from specially created LXC containers. Each LXC container attacked the specified hypervisor and the Ubuntu Apache Web server. <br>
The organizers managed the DoS attack from a separate virtual machine with the open-source automation server Jenkins installed. Using Jenkins, for each student group, it was possible to configure individual automatically executed scripts to perform different scenarios and set attack parameters, e.g. at specific and predefined time intervals.<br>
The Hping3 network tool was used to simulate a web server SYN Flood Attack---the most common and effective way to attack a Web server and make its services unavailable. The attack also made the entire virtual machine network adapter and all protocols unavailable. The open-source process supervision tool Monit enabled monitoring of the progress of the attack and the effectiveness of blocking. <br>
The architecture is a completely isolated environment and could not harm the external infrastructure of the university. Users could access it from any place via the Internet, and the attack automatization enabled the implementation of dynamic scenarios. <br>
# 2. Technical implementation
This section will cover the technical steps to create the environment for the task.
## 2.1. Installing the hypervisor
One of the hypervisors must be installed to reproduce the virtualization environment. In this case is used Proxmox VE, tutorial on how to install the Proxmox VE virtualization environment: https://pve.proxmox.com/pve-docs/chapter-pve-installation.html

## 2.2. Deploy Jenkins LXC
The Jenkins script execution management platform was used to create the script execution environment. This platform was installed as a Proxmox LXC container from Turnkey (https://www.turnkeylinux.org). For the LXC container was allocated the following computing resources: 4 CPUs, 8 GB RAM, 50 GB HDD.  <br>
How to create LXC in Proxmox: https://www.tecmint.com/proxmox-create-container/  <br>
Jenkins LXC overview: https://www.turnkeylinux.org/jenkins  <br>

## 2.3. Deploy base Ubuntu 22.04 Desktop virtual machines
Create Ubuntu 22.04 Desktop virtual machine for each group of students. Since these machines will all be the same, we can use Linked Clone technology (https://pve.proxmox.com/wiki/VM_Templates_and_Clones). 
<br> Create a basic Ubuntu 22.04 Desktop virtual machine by assigning the following computing resources: 2 CPUs, 4GB RAM, 30 - 50 GB HDD. Here's a tutorial on how to create an Ubuntu 22.04 Desktop virtual machine: https://credibledev.com/creating-a-ubuntu-vm-on-proxmox/  <br>
When the machine is created, install the Wireshark open-source packet analyzer. How to Install Wireshark on Ubuntu 22.04: https://www.cherryservers.com/blog/install-wireshark-ubuntu 
<br> Install the Apache Web server (https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-22-04) and install one of the freely available HTML page templates (https://www.free-css.com/free-css-templates). You can customize the HTML template for your needs.<br>

After the base machine is prepared, convert it to Proxmox Template and create as many Linked Clone virtual machines as there will be student teams. Proxmox offers a utility called pveperf that allows you to create clone copies of a link. To create multiple clone machines, you can use this utility via shell. For example, the following command will create 10 link clone copies of the original machine named "originalvm":
<br><br>
`for i in {1..10}; do qm clone originalvm $i --name linkedclone$i; done`
<br><br>
In this command, originalvm is the ID of the original machine and linkedclone$i is the name of each new clone machine, where $i is a number between 1 and 10. For higher-level automation, this can be incorporated into the Ansible or Terraform code execution process.

## 2.4. Deploy DoS attack LXC instances. 
Since each studnet group must ensure that the DoS attack was carried out by an individual instance and IP address, an appropriate amount of LXC instances must be created. To automatically create 10 LXC containers in Proxmox, a script can be created that uses the Proxmox API to perform this task. Here is an example of how this could be done using a bash script and curl to communicate with the Proxmox API: <br><br>
`
#!/bin/bash
PROXMOX_HOST="your_proxmox_host"
USERNAME="your_username"
PASSWORD="your_password"
NODE="your_node_name"
API_URL="https://$PROXMOX_HOST/api2/json"
for i in {1..10}; do
    curl -k -d "vmid=$i&ostemplate=local:vztmpl/debian-10-turnkey-openvpn_16.0-1_amd64.tar.gz&hostname=lxc$i&password=your_container_password" -H "CSRFPreventionToken: " -b "PVEAuthCookie=$(curl -sk -d "username=$USERNAME&password=$PASSWORD" $API_URL/access/ticket | awk -F'"' '{print $2}')" $API_URL/nodes/$NODE/lxc;
done
`
<br><br>
Before running this script, please replace your_proxmox_host, your_username, your_password, your_node_name and your_container_password with the relevant Proxmox server parameters and your preferences. This script will create 10 LXC containers using the available Debian 10 OS image. You can also change this image to another one if you want.<br>
Important: Before running this script, make sure your Proxmox server is configured and ready to communicate with the API, and make sure your data and passwords are transmitted securely, for example over a secure connection (HTTPS). Also, before committing to mass operations, it's always wise to test them first in environments where bugs are not as critical.


## 2.5. Create Jenkins freestyle project
Create a Jenkins freestyle project and add the ability to connect to each of the Debian 10 LXC containers using SSH. Configuring SSH connection to a remote host in Jenkins (SSH-plugin): https://medium.com/cloudera-devops-beyond/configuring-ssh-connection-to-remote-host-in-jenkins-ssh-plugin-e2e9a00559f1 <br> Once this is done, use the Jenkins Schedule functionality: https://www.cloudbees.com/blog/how-to-schedule-a-jenkins-job
<br>Add the execution code to the Jenkins project and configure it so that it "attacks" the Apache Web server of the Ubuntu 22.04 Desktop virtual machine. <br>

`#!/bin/bash
timeout 60s hping3 -S -p 80 -i u1 --flood --syn -d 1200 -w 640 10.15.27.57
exit 0 `

<br>

DOS Flood With hping3: https://linuxhint.com/hping3/><br>
How To Perform TCP SYN Flood DoS Attack & Detect It With Wireshark - Kali Linux Hping3: https://www.firewall.cx/tools-tips-reviews/network-protocol-analyzers/performing-tcp-syn-flood-attack-and-detecting-it-with-wireshark.html
## 2.6. Monitoring student progress
From an additional virtual machine where the Monit monitoring system is installed (https://www.atlantic.net/vps-hosting/how-to-install-monit-monitoring-server-on-ubuntu/), the status of all Apache Web servers is monitored. Connect each Ubuntu 22.04 Desktop Apache Web server to the Monit system and monitor its status (https://mmonit.com/monit/documentation/monit.html#HTTP) <br> 
A group of students must block the IP address of the attacker's LXC container. Once this is done successfully, the Ubuntu 22.04 Desktop virtual machine Apache Web server with HTTML webpage will be available in the Monit system as well during the Jenkins script attack.
