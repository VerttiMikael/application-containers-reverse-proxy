# application-containers-reverse-proxy
This project involves using Portainer Docker and Nginx reverse proxy to create two Web services on a single virtual machine and IP. The virtual machine is created with Terraform and in CSC cPouta and we will also store Docker files on a persistent volume for good practise.

## Prerequisites:
- Linux operating system or local Ubuntu vm
- ssh keypair for the VM instances
---

We will try to achieve architecture like this:
![Projects_Pics](/Project_Pics/image16.png)

In this project we will reuse the Terrafrom code from my previous repo: cloud-infra-terraform-ansible. The providers.tf stays the same but we need to modify the main.tf file.

## Setting up terraform

### Step 1: Modify the main.tf file
```bash
nano main.tf

# main.tf

resource "openstack_compute_keypair_v2" "my-cloud-key" {  
  name       = "project1-key"  
  public_key = "ssh-ed25519 AAA....”  
}  

# Create the virtual machine   
resource "openstack_compute_instance_v2" "VM1" {  
  name          = "VM 1"  
  image_name      = "Ubuntu-22.04"  
  flavor_name     = "standard.small"  
  network {  
    name = "project_20118XX"  
  }  
  security_groups = [openstack_compute_secgroup_v2.secgroup_1.name]  
  key_pair ="${ openstack_compute_keypair_v2.my-cloud-key.name}"  
}   

# Implement the floating ip to VM 1  
resource "openstack_networking_floatingip_v2" "fip_1" {  
  pool = "public"  
}  

resource "openstack_compute_floatingip_associate_v2" "fip_1" {  
  floating_ip = openstack_networking_floatingip_v2.fip_1.address  
  instance_id = openstack_compute_instance_v2.VM1.id  
}  

# Creating the sec group for VM1  
resource "openstack_compute_secgroup_v2" "secgroup_1" {  
  name        = "secgroup_1"  
  description = "security group for VM1"  

  rule {  
    from_port   = 22  
    to_port     = 22  
    ip_protocol = "tcp"  
    cidr        = "0.0.0.0/0"  
  }  

  rule {  
    from_port   = 80  
    to_port     = 80  
    ip_protocol = "tcp"  
    cidr        = "0.0.0.0/0"  
  }  

  rule {  
    from_port   = 81  
    to_port     = 81  
    ip_protocol = "tcp"  
    cidr        = "0.0.0.0/0"  
  }

  rule {  
    from_port   = 9443  
    to_port     = 9443 
    ip_protocol = "tcp"  
    cidr        = "0.0.0.0/0"  
  }  
```

### Step 2: Save and deploy the virtual machine
```bash
terraform apply 
```

## Persistent volume setup

We want to store all Docker files in a persistent volume so that if the virtual machine is deleted, our images, containers, and volumes will be preserved.

### Step 1: Create the persistent volume and attach it to the VM
```bash
openstack volume create --description 'Docker Project' --size 5 new_volume

openstack server add volume 'VM 1' new_volume 
```

### Step 2: Create a file system and mount it
```bash
sudo mkfs.xfs /dev/vdb

sudo mkdir -p /media/volume

sudo mount /dev/vdb /media/volume

sudo chown ubuntu:ubuntu /media/volume
```

## Portainer Docker setup

### Step 1: Install Docker engine using the convenience script
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh 
```

### Step 2: Add your user to the Docker group
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

### Step 3: Install Docker compose and set up the docker-compose.yml file
```bash
mkdir -p ~/.docker/cli-plugins/ 
curl -SL https://github.com/docker/compose/releases/download/v2.32.1/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose

chmod +x ~/.docker/cli-plugins/docker-compose

nano docker-compose.yml

# docker-compose.yml

services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

### Step 4: Change the default Docker directory to the persistent volume

First lets create a configuration file to tell the docker daemon what is the location of the data directory  
```bash
sudo nano /etc/docker/docker.json
# docker.json
{ 
   "data-root": "/media/volume/docker" 
}
```

Then we can copy the data to the new directory and also rename the old directory to avoid conflict

```bash
sudo rsync -aP /var/lib/docker/ "/media/volume/docker"

sudo mv /var/lib/docker /var/lib/docker.old 
```

### Step 5: Install Portainer
```bash
docker volume create portainer_data

docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:2.21.5
```

### Step 6: Open Portainer Docker WebUI and login
```bash
https://86.50.YYY.XXX:9443
```


## Nginx Proxy Manager setup

### Step 1: Create a new stack using the Portainer web editor

Name the stack nginx-proxy-manager-stack

Paste the docker-compose.yml file into the text box and deploy the stack

### Step 2: Login to the Nginx Proxy Manager WebUI

```bash
https://86.50.YYY.XXX:81
```

### Step 3: Create the new containers in the Portainer WebUI

Name the containers site1_simple and site2_simple. 

Use the images nginxdemos/hello and bsord/tetris

From the Network tab, select "nginx-proxy-manager-stack_default"

### Step 4: Create proxy hosts for the containers

Name the Domain names simple.example.com and simple.example2.com 

Use the created container names in the Forward Hostname

Forward Port is 80

### Step 4: Add the domain lines to your workstations hosts file and specify the public IP of your VM

```bash
sudo nano /etc/hosts

# hosts

127.0.0.1 localhost 
127.0.1.1 verlep-VirtualBox 

# The following lines are desirable for IPv6 capable hosts 
::1     ip6-localhost ip6-loopback 
fe00::0 ip6-localnet 
ff00::0 ip6-mcastprefix 
ff02::1 ip6-allnodes 
ff02::2 ip6-allrouters 
86.50.YYY.XXX simple.example.com 
86.50.YYY.XXX simple.example2.com 
```


# Results

We should now have two working Web services with one being a tetris game and the other a simple nginx web page



