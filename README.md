# Setup IPFS and Host Simple Benign HTML and Executable

> Austin Lai | August 22nd, 2022

---

## Table of Contents

<!-- TOC -->

- [Setup IPFS and Host Simple Benign HTML and Executable](#setup-ipfs-and-host-simple-benign-html-and-executable)
    - [Table of Contents](#table-of-contents)
    - [Description or Introduction](#description-or-introduction)
        - [Learn how IPFS works](#learn-how-ipfs-works)
        - [IPFS Setup Environment](#ipfs-setup-environment)
        - [IPFS Network Port Requirement](#ipfs-network-port-requirement)
    - [Setup IPFS in AWS Environment](#setup-ipfs-in-aws-environment)
        - [Create EC2 Instance in AWS](#create-ec2-instance-in-aws)
        - [Setup IPFS](#setup-ipfs)
            - [Publishing IPNS](#publishing-ipns)
    - [Setup with Terraform IaC](#setup-with-terraform-iac)
        - [Preparation of Terraform for IPFS on AWS EC2](#preparation-of-terraform-for-ipfs-on-aws-ec2)
        - [Terraform Project Folder Structure](#terraform-project-folder-structure)
            - [AWS KEY for Terraform](#aws-key-for-terraform)
            - [Terraform Code - main.tf](#terraform-code---maintf)
            - [IPFS Script](#ipfs-script)
                - [Root version](#root-version)
                - [Ubuntu user version](#ubuntu-user-version)
        - [Running Terraform](#running-terraform)
    - [Additional Information](#additional-information)
        - [IPFS Garbage Collection](#ipfs-garbage-collection)
        - [Content of IPFS Hosted Locally is publicly available](#content-of-ipfs-hosted-locally-is-publicly-available)
        - [Additional Article Reference](#additional-article-reference)

<!-- /TOC -->

## Description or Introduction

<!-- Description -->

What is IPFS?

IPFS ([the InterPlanetary File System](https://docs.ipfs.tech/concepts/what-is-ipfs/)) is a peer-to-peer hypermedia distribution protocol addressed by content and identities.

IPFS also a **distributed file system** for storing and accessing files, websites, applications, and data that seeks to connect all computing devices with the same system of files. In some ways, this is similar to the original aims of the Web, but IPFS is actually more similar to a single BitTorrent swarm exchanging Git objects.

<br>

### Learn how IPFS works

To learn more about how IPFS works, explore the following resources:

- Refer to <https://github.com/ipfs/ipfs>
- [IPFS Docs: How IPFS Works](https://docs.ipfs.tech/concepts/how-ipfs-works)
- [IPFS Specifications](https://github.com/ipfs/specs)

<br>

### IPFS Setup Environment

IPFS from <https://docs.ipfs.tech/concepts/what-is-ipfs/#decentralization> can be installed and host in below environment:

- Local device
- Virtual Machines
- Cloud environment (AWS, GCP, and etc.)
- Free services such as <https://nft.storage/files/> and <https://web3.storage/>, **BEWARE!!! ALL DATA UPLOADED TO FREE SERVICES WILL BE PERMANENT AVAILABLE TO PUBLIC!!!**

For this example, we are going to setup IPFS in AWS environment. If you are familiar with AWS environment, you may skip [Create EC2 Instance in AWS](#create-ec2-instance-in-aws) and refer to [Setup IPFS](#setup-ipfs).

<!-- /Description -->

<br>

### IPFS Network Port Requirement

- 4001-4002 - default libp2p swarm port - should be open to public for all nodes if possible.
- 5001 - API port - provides write/admin access to the node, shouldn't be exposed at all.
- 8080 - Gateway + read only API subset - quite safe to expose, but operating public gateway may still be a risk in some ways.
- 8081 - Websocket transfer. However, this isn't enabled by default.

<br>

## Setup IPFS in AWS Environment

### Create EC2 Instance in AWS

Go to <https://aws.amazon.com/free/> and sign-up with free account.

Once done, go to "EC2".

![AWS-EC2.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2.png)

Then in the dashboard, select "Launch Instance"

![AWS-EC2-LaunchInstance.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2-LaunchInstance.png)

Next, enter naming for the instance.

![AWS-EC2-LaunchInstance-Naming.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2-LaunchInstance-Naming.png)

And carefully check if you select "Free-Tier" Ubuntu Server OS.

![AWS-EC2-LaunchInstance-SelectOSImage.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2-LaunchInstance-SelectOSImage.png)

Next, "Generate SSH Key".

![AWS-EC2-LaunchInstance-FreeTier-SSHKey.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2-LaunchInstance-FreeTier-SSHKey.png)

Then, go to "Network Setting".

Setup Security Group Rule to allow SSH access from your public IP instead of open-up with "0.0.0.0/0"

![AWS-EC2-LaunchInstance-NetworkSetting.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2-LaunchInstance-NetworkSetting.png)

Then we add 2 new security group rules as below:

![AWS-EC2-LaunchInstance-NetworkSetting-CustomRules.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2-LaunchInstance-NetworkSetting-CustomRules.png)

**Remember to provide informative naming to the rules.**

Before proceed to create the instance, we can add additional volume to the instance since free tier given 30GB for the allowance of storage.

![AWS-EC2-LaunchInstance-Storage.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2-LaunchInstance-Storage.png)

Now, we are good to go to create the instance with above configuration.

Once, it is up and running; we are good to proceed next step, you may connect to the instance via SSH:

```bash
ssh -i ipfs.pem ubuntu@ec2-x-x-x-x.us-xxxx-2.compute.amazonaws.com
```

### Setup IPFS

Go to <https://dist.ipfs.tech/#go-ipfs> to get the latest IPFS binary available.

To install IPFS in the SSH session, you can use the following commands:

```bash
wget https://dist.ipfs.tech/kubo/v0.14.0/kubo_v0.14.0_linux-amd64.tar.gz
tar xvfz kubo_v0.14.0_linux-amd64.tar.gz
rm kubo_v0.14.0_linux-amd64.tar.gz 
cd kubo
sudo bash ./install.sh
```

Next, setup directory for IPFS and initialize IPFS with the following:

```bash
echo 'export IPFS_PATH=/data/ipfs' >> ~/.bash_profile
source ~/.bash_profile
sudo mkdir -p $IPFS_PATH
sudo chown ubuntu:ubuntu $IPFS_PATH
ipfs init -p server
ipfs config Datastore.StorageMax 8GB

# Configure IPFS to allow direct access to the instance's gateway
ipfs config Addresses.Gateway /ip4/0.0.0.0/tcp/8080
```

Then, we create IPFS services as below:

```bash
sudo bash -c 'cat >/lib/systemd/system/ipfs.service <<EOL
[Unit]
Description=ipfs daemon
[Service]
ExecStart=/usr/local/bin/ipfs daemon --enable-gc
Restart=always
User=ubuntu
Group=ubuntu
Environment="IPFS_PATH=/data/ipfs"
[Install]
WantedBy=multi-user.target
EOL'
```

And then enable IPFS and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ipfs.service
sudo systemctl start ipfs
sudo systemctl status ipfs
```

Once, the IPFS service is running; we can use command below to check the IPFS peers:

```bash
ipfs swarm peers
```

We have now created a EC2 instance with Ubuntu Server, installed IPFS on it, and started running IPFS daemon.

We can opened up IPFS gateway by browse to its address in a browser, <http://ec2-xx-xxx-xxx-xx.us-xxxx-2.compute.amazonaws.com:8080/ipfs/HASH-KEY>.

We should be able to view the IPFS default pinning files:

![AWS-EC2-IPFS-Web.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/AWS-EC2-IPFS-Web.png)

Then, we can try to host multi-page sites from <https://docs.ipfs.tech/how-to/websites-on-ipfs/multipage-website/#multi-page-website>

Once, we reach to the step of **Add files to IPFS**, we can use command below to add files since the guide using "IPFS Desktop".

```bash
cd multi-page-first-step
ipfs add -r .
```

Once done, we may visit to the site where the HASH-KEY provided when you add files to IPFS.

![ipfs-multi-page-sites.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/ipfs-multi-page-sites.png)

<br>

#### Publishing IPNS

Using CIDs to get content is great; it means that the user always gets the content that they want. But what if the user doesn't know what they're looking for and just wants the latest version of that content? This is where **IPNS** comes in handy.

Instead of sharing the CID of the website, we publish the root CID of the website to IPNS and then share the key we get from IPNS.

```bash
ubuntu@ip:cd multi-page-first-step

ubuntu@ip:~/multi-page-first-step$ ipfs add -r .
added Qme17nwco6bCWzhxfnxure59nqsVjvZkip4bCTW5XBovJe multi-page-first-step/about.html
added QmSydH6jDbDMxj9F9YZmBtD5TKpgZNM5P1LXkkK83fQimz multi-page-first-step/index.html
added QmW8U3NEHx3p73Nj9645sGnGa8XzR43rQh3Kd52UKncWMo multi-page-first-step/moon-logo.png
added QmYk417i1HxpHirBS79vZsbLJNpqQhMzEELpgkGAAXajRu multi-page-first-step/screen.exe
added QmSYaaaVKea6bbbgzW4bg6x5viJJYwsDLqcHksUo59PM4q multi-page-first-step

ubuntu@ip:~/multi-page-first-step$ ipfs name publish /ipfs/QmSYaaaVKea6bbbgzW4bg6x5viJJYwsDLqcHksUo59PM4q
Published to k51qzi5uqu5dhvou4mt0gs95jht2pstdq9qmz098iodly6gcp2b7pob7wb33g3: /ipfs/QmSYaaaVKea6bbbgzW4bg6x5viJJYwsDLqcHksUo59PM4q

ubuntu@ip:~/multi-page-first-step$
```

The `k51qzi...` is the IPFS installation's key! This is what we can use to point people to the content.

We should now be able to view it by going to `http://ec2-xx-xxx-xxx-xx.us-xxxx-2.compute.amazonaws.com:8080/ipns/k51qzi....`. Replace `k51qzi...` with the output from the previous step.

![ipns-sites.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/ipns-sites.png)

**Whenever you make any changes to your project, simply re-add your content to IPFS and publish it to IPNS**

<br>

## Setup with Terraform IaC

### Preparation of Terraform for IPFS on AWS EC2

Now, we can integrate the IPFS setup on AWS EC2 instance with Terraform IaC.

There are a few things we need prepare before we start.

- Download terraform to your machines, you can find the suitable terraform version from <https://www.terraform.io/downloads>
    - For windows device, you may also use `choco install terraform`
- AWS Access Key and AWS Secret Key, you can read more from <https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html>
- AWS CLI installed, you can read more from <https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html>

<br>

### Terraform Project Folder Structure

First, create a folder for serving as Terraform project which will be used to build terraform code.

The folder will be at least consist below:

![tree-view.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/tree-view.png)

<br>

#### AWS KEY for Terraform

`aws.key` is where we store the AWS access key and AWS secret access key, that will be used by terraform to connect to AWS and provision AWS EC2 instance.

The `aws.key` format as below:

```text
[ASSIGN-A-PROFILE-NAME]
aws_access_key_id = YOUR-AWS-ACCESS-KEY-ID
aws_secret_access_key = YOUR-AWS-SECRET-ACCESS-KEY
```

The **Profile Name** can be anything.

#### Terraform Code - main.tf

`main.tf` will be the terraform code, however, if you prefer; you may split the code to multiple file, e.g. `variable.tf`, `profile.tf`, `instance.tf` and etc.

```terraform
# Setting up Terraform AWS Provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }
  required_version = ">= 1.2.0"
}
provider "aws" {
  region                   = "us-east-1"
  shared_credentials_files = ["aws.key"]
}

# When running terraform, it will prompt to you to provide SSH Key Name
variable "key_name" {}

# When running terraform, it will prompt to you to provide AWS EC2 Name
variable "tag_name" {}

# Create SSH Public and Private key pair based on the Key Name provided
resource "aws_key_pair" "ipfs-host" {
  key_name   = var.key_name
  public_key = tls_private_key.ipfs-host.public_key_openssh
}
resource "tls_private_key" "ipfs-host" {
  algorithm = "RSA"
  rsa_bits  = 4096
}
resource "local_file" "ipfs-host" {
  content  = tls_private_key.ipfs-host.private_key_pem
  filename = "ipfs-host.pem"
}

# Creating AWS EC2 Instance
resource "aws_instance" "ipfs-host" {
  
  # ami is the AWS EC2 Instance image ID, we are using Ubuntu Server image here
  ami                    = "ami-052efd3df9dad4825"
  availability_zone      = "us-east-1d"
  ebs_optimized          = false
  instance_type          = "t2.micro"
  monitoring             = false
  key_name               = aws_key_pair.ipfs-host.key_name
  vpc_security_group_ids = ["sg-03210c620abc6f04b"]
  source_dest_check      = true
  tags = {
    Name = var.tag_name
  }

  # We are adding 30Gb of EBS storage for IPFS
  ebs_block_device {
    device_name           = "/dev/sdb"
    snapshot_id           = ""
    volume_type           = "gp3"
    volume_size           = 30
    delete_on_termination = true
  }

  # This is the default root storage
  root_block_device {
    volume_type           = "gp2"
    volume_size           = 8
    delete_on_termination = true
  }

  # Here we input our script to setup IPFS
  # BEWARE ! The script will run under root permission !
  user_data = "${file("root-script.sh")}"

  # You may also use this section to setup IPFS with the script
  # Uncomment this to use this section
  # provisioner "file" {
  #   source      = "ubuntu-script.sh"
  #   destination = "/tmp/ubuntu-script.sh"
  # }
  # provisioner "remote-exec" {
  #   inline = [
  #     "sudo chmod +x /tmp/ubuntu-script.sh",
  #     "sudo /tmp/ubuntu-script.sh"
  #   ]
  # }

  # Setup SSH connection to newly created AWS EC2 instance with the SSH Key generated earlier
  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = tls_private_key.ipfs-host.private_key_pem
    host        = aws_instance.ipfs-host.public_ip
  }
}
```

<br>

#### IPFS Script

<br>

##### Root version

```bash
#!/bin/bash
{
    set -x

    # Download kubo-ipfs into tmp folder
    # unzip and install
    # remove leaft-over files after install
    cd /tmp/
    wget https://dist.ipfs.tech/kubo/v0.14.0/kubo_v0.14.0_linux-amd64.tar.gz
    tar xvfz kubo_v0.14.0_linux-amd64.tar.gz
    rm kubo_v0.14.0_linux-amd64.tar.gz 
    bash ./kubo/install.sh
    rm -rf kubo
    
    # Setup IPFS Path
    bash -c 'echo export IPFS_PATH=/data/ipfs >> /etc/profile' 
    source /etc/profile

    # Setup IPFS Path
    echo 'export IPFS_PATH=/data/ipfs' >> /home/ubuntu/.bash_profile
    source /home/ubuntu/.bash_profile
    
    # Confirm IPFS Path and ensure ubuntu user home directories with its ownership
    echo $IPFS_PATH > /tmp/IPFS_PATH.txt
    chown -R ubuntu:ubuntu /home/ubuntu

    # Create IPFS Data Directory with the define IPFS Path
    # Set the correct ownership of the IPFS Data Directory
    mkdir -p $IPFS_PATH
    chown ubuntu:ubuntu $IPFS_PATH

    # Initial IPFS as server
    cd $IPFS_PATH
    ipfs init -p server

    # Configure IPFS Storage Size
    ipfs config Datastore.StorageMax 8GB

    # Configure IPFS to allow direct access to the instance's gateway
    ipfs config Addresses.Gateway /ip4/0.0.0.0/tcp/8080

    # Once again, set the correct ownership of the IPFS Data Directory
    chown -R ubuntu:ubuntu $IPFS_PATH

    # Create IPFS Service
    echo '[Unit]' | tee /lib/systemd/system/ipfs.service
    echo 'Description=ipfs daemon' | tee -a /lib/systemd/system/ipfs.service
    echo 'After=network.target' | tee -a /lib/systemd/system/ipfs.service
    echo '[Service]' | tee -a /lib/systemd/system/ipfs.service
    echo 'ExecStart=/usr/local/bin/ipfs daemon --enable-gc' | tee -a /lib/systemd/system/ipfs.service
    echo 'StandardOutput=journal' | tee -a /lib/systemd/system/ipfs.service
    echo 'KillSignal=SIGINT' | tee -a /lib/systemd/system/ipfs.service
    echo 'Restart=always' | tee -a /lib/systemd/system/ipfs.service
    echo 'User=ubuntu' | tee -a /lib/systemd/system/ipfs.service
    echo 'Group=ubuntu' | tee -a /lib/systemd/system/ipfs.service
    echo 'Environment="IPFS_PATH=/data/ipfs"' | tee -a /lib/systemd/system/ipfs.service
    echo '[Install]' | tee -a /lib/systemd/system/ipfs.service
    echo 'WantedBy=multi-user.target' | tee -a /lib/systemd/system/ipfs.service

    # Reload system service and enabled boot-start for IPFS
    systemctl daemon-reload
    systemctl enable ipfs.service
    systemctl start ipfs
    systemctl status ipfs

    # Check if IPFS is running and connected to peers
    ipfs swarm peers
    
    set +x

} 2>&1 | tee $(whoami)-script.log
```

<br>

##### Ubuntu user version

```bash
#!/bin/bash
{
    set -x

    # Download kubo-ipfs into tmp folder
    # unzip and install
    # remove leaft-over files after install
    cd /tmp/
    wget https://dist.ipfs.tech/kubo/v0.14.0/kubo_v0.14.0_linux-amd64.tar.gz
    tar xvfz kubo_v0.14.0_linux-amd64.tar.gz
    rm kubo_v0.14.0_linux-amd64.tar.gz 
    sudo bash ./kubo/install.sh
    rm -rf kubo
    
    # Setup IPFS Path
    sudo bash -c 'echo export IPFS_PATH=/data/ipfs >> /etc/profile' 
    source /etc/profile
    
    # Setup IPFS Path
    echo 'export IPFS_PATH=/data/ipfs' >> /home/ubuntu/.bash_profile
    source /home/ubuntu/.bash_profile
    
    # Confirm IPFS Path and ensure ubuntu user home directories with its ownership
    echo $IPFS_PATH > /tmp/IPFS_PATH.txt

    # Create IPFS Data Directory with the define IPFS Path
    # Set the correct ownership of the IPFS Data Directory
    sudo mkdir -p $IPFS_PATH
    sudo chown ubuntu:ubuntu $IPFS_PATH

    # Initial IPFS as server
    cd $IPFS_PATH
    ipfs init -p server

    # Configure IPFS Storage Size
    ipfs config Datastore.StorageMax 8GB

    # Configure IPFS to allow direct access to the instance's gateway
    ipfs config Addresses.Gateway /ip4/0.0.0.0/tcp/8080

    # Once again, set the correct ownership of the IPFS Data Directory
    sudo chown -R ubuntu:ubuntu $IPFS_PATH

    # Create IPFS Service
    echo '[Unit]' | sudo tee /lib/systemd/system/ipfs.service
    echo 'Description=ipfs daemon' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'After=network.target' | sudo tee -a /lib/systemd/system/ipfs.service
    echo '[Service]' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'ExecStart=/usr/local/bin/ipfs daemon --enable-gc' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'StandardOutput=journal' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'KillSignal=SIGINT' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'Restart=always' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'User=ubuntu' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'Group=ubuntu' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'Environment="IPFS_PATH=/data/ipfs"' | sudo tee -a /lib/systemd/system/ipfs.service
    echo '[Install]' | sudo tee -a /lib/systemd/system/ipfs.service
    echo 'WantedBy=multi-user.target' | sudo tee -a /lib/systemd/system/ipfs.service

    # Reload system service and enabled boot-start for IPFS
    sudo systemctl daemon-reload
    sudo systemctl enable ipfs.service
    sudo systemctl start ipfs
    sudo systemctl status ipfs
    
    # Check if IPFS is running and connected to peers
    ipfs swarm peers
    
    set +x

} 2>&1 | tee $(whoami)-script.log
```

<br>

### Running Terraform

After all the require files ready, we are good to proceed and run terraform.

First, we run `terraform init`, to initialize the environment and download necessary terraform files and provider.

Next, we can run `terraform fmt` to format correctly our `main.tf` and `terraform validate` to validate the code of `main.tf`

Then, we run `terraform plan --out NAME.tfplan`. This will create terraform plan template for use to execute it, this is a good practice to prevent unintentional destroy or commit the code after changes.

When everything is ready, we are good to execute the terraform plan by running the command `terraform apply "NAME.tfplan"`.

![terraform-ipfs-aws-ec2-creation.gif](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/terraform-ipfs-aws-ec2-creation.gif)

Once terraform finish executed.

We can check on AWS Console and we are ready to connect via SSH as shown below:

![connect-to-aws-ec2-and-check-ipfs.gif](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/connect-to-aws-ec2-and-check-ipfs.gif)

If you want to terminated or destroy the EC2 Instance, you can run `terraform destroy`

![terraform-destroy.gif](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/terraform-destroy.gif)

<br>

## Additional Information

### [IPFS Garbage Collection](https://docs.ipfs.tech/concepts/persistence/#garbage-collection)

Garbage collection is a form of automatic resource management widely used in software development. The garbage collector attempts to reclaim memory occupied by objects that are no longer in use. IPFS uses garbage collection to free disk space on your IPFS node by deleting data that it thinks is no longer needed.

The IPFS garbage collector is configured in the `Datastore` section of the `Kubo` config file. The important settings related to the garbage collector are:

- `StorageGCWatermark`: The percentage of the `StorageMax` value at which a garbage collection will be triggered automatically, if the daemon is running with automatic garbage collection enabled. The default is `90`.
- `GCPeriod`: Specify how frequently garbage collection should run. Only used if automatic garbage collection is enabled. The default is `1 hour`.

To manually start garbage collection, run `ipfs repo gc`.

To enable automatic garbage collection use `--enable-gc` when starting the IPFS daemon:

```
ipfs daemon --enable-gc
```

A nice reference article with regards to IPFS Garbage Collection can be found at <https://blog.logrocket.com/guide-ipfs-garbage-collection/>

### Content of IPFS Hosted Locally is publicly available

Content of IPFS Hosted Locally is publicly available through IPFS Public gateways which you can find from <https://docs.ipfs.tech/concepts/ipfs-gateway/#gateway-providers>.

This process is the same as using any other IPFS gateway -- only the address of the gateway is different.

If you're using the hash of a specific snapshot of content, use the path `https://ipfs.io/ipfs/<your-ipfs-hash>`. If you're using an IPNS hash to get the latest version of some content, use the path `https://ipfs.io/ipns/<your-ipns-hash>`.

Below is the content access from local IPFS gateway:

![ipns-sites.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/ipns-sites.png)

Below is the content access from public IPFS gateway:

![content-from-public-ipfs.png](https://github.com/austin-lai/Setup-IPFS-and-Host-Simple-Benign-HTML-and-Executable/blob/master/content-from-public-ipfs.png)

Detail article can be found at <https://ipfs.io/ipfs/QmcqCBTktATA1gzLuR5RcbXRZ375uXtpsioY7rSFhzCCZy/classical-web/lessons/public-gateways.html>.

<br>

### Additional Article Reference

<https://blog.ipfs.tech/2022-06-09-practical-explainer-ipfs-gateways-1/>

<https://blog.ipfs.tech/2022-06-30-practical-explainer-ipfs-gateways-2/>

<br />

---

> Do let me know any command or step can be improve or you have any question you can contact me via THM message or write down comment below or via FB
