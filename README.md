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
