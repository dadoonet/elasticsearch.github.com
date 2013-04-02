---
layout: tutorial
title: elasticsearch on Windows Azure
cat: tutorials
author: David Pilato
nick:
tutorial_desc: Running elasticsearch on Windows Azure
---

elasticsearch on Windows Azure
==============================

The Windows Azure platform provides 2 different kind of services for running elasticsearch:

* Virtual machines
* Cloud services

This tutorial will describe the needed steps to have full up and running elasticsearch cluster.

Using Windows Azure Virtual Machines
====================================

When using Windows Azure Virtual Machines (VMs), you can have two strategies:

* Running everything behind a single network virtual port
* Running standalone VMs

For the first one, only VM behind this same virtual port can talk to each other. That means that with this mode, you can use elasticsearch default multicast discovery to build a cluster.

If you want to have standalone VMs, you will can use Unicast settings to define all your nodes or best,
you can use the elasticsearch-cloud-azure plugin to let it fetch information about your nodes using
azure API.


Using a single network port
---------------------------

TODO

Using elasticsearch cloud Azure plugin
--------------------------------------

### Create Azure VMs

First, you have to create your [Windows Azure account](http://www.windowsazure.com/) and connect 
to [Management Console](https://manage.windowsazure.com/).

Create your first VM by clicking on `+ NEW` button. It will open the following window:

![Create VM assistant](../images/azure/azure-create-vm0.png "Create VM assistant")

Choose `Compute` -> `Virtual Machine` -> `From Gallery`. You will be asked to choose your operating system.

![Create VM assistant](../images/azure/azure-create-vm1.png "Create VM assistant")

Once done, go to next step and choose settings for your VM:

* Virtual machine name
* User name (will be used to log in using SSH later)
* Password (password for this user)
* Size (size of the box: extra small, small...)

Check the `Upload SSH key...` box and upload your PEM Azure certificate. See this 
[guide](http://www.windowsazure.com/en-us/manage/linux/how-to-guides/ssh-into-linux/) to have
details on how to create keys for Azure.

![Create VM assistant](../images/azure/azure-create-vm2.png "Create VM assistant")

*Note for Mac users, if you don't succeed in clicking on the upload certificate button, just select it
and press the space bar. It will display the upload window...*

Once done, go to next step, select `Standalone virtual machine` and choose a DNS name for your VM:

![Create VM assistant](../images/azure/azure-create-vm3.png "Create VM assistant")

Once done, go to next step and finally valid your instance creation:

![Create VM assistant](../images/azure/azure-create-vm4.png "Create VM assistant")

### Network settings

Note that by default, your SSH is not bound to port 22. You may want to modify that by entering in
your VM configuration and click on `Endpoints` tab.
Modify it if you need:

![SSH Settings](../images/azure/azure-check-ssh.png "SSH Settings")

Also, you need to add a rule and open your 9300 port.

![SSH Settings](../images/azure/azure-check-transport.png "SSH Settings")


### Install elasticsearch

You will need to use your azure key from your VM, so you need to copy the private key and the certificate
on your platform, let's say in `/home/elasticsearch/keys` dir.

```sh
# Create a dir for your keys
ssh -i azurePrivateKey.key elasticsearch@es-vm2.cloudapp.net 'mkdir /home/elasticsearch/keys'

# Transform private key to PEM format
openssl pkcs8 -topk8 -nocrypt -in azurePrivateKey.key -inform PEM -out azure.pem -outform PEM
# Transform certificate to PEM format
openssl x509 -inform der -in azureCert.cer -out azurecert.pem

# Copy your azure private key and certificate
scp -i azurePrivateKey.key azure.pem azurecert.pem elasticsearch@es-vm2.cloudapp.net:/home/elasticsearch/keys
```

Connect with SSH to your VM:

```sh
ssh -i azurePrivateKey.key elasticsearch@es-vm2.cloudapp.net
```

Install elasticsearch:

```sh
# Download the .deb package
curl https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-0.20.6.deb -o elasticsearch-0.20.6.deb

# Install it
sudo apt-get update
sudo dpkg -i elasticsearch-0.20.6.deb
sudo apt-get -f install
```

Check that elasticsearch is running:

```sh
curl http://localhost:9200/
```

This command should give you a JSON result:

```javascript
{
  "ok" : true,
  "status" : 200,
  "name" : "Lunatica",
  "version" : {
    "number" : "0.20.6",
    "snapshot_build" : false
  },
  "tagline" : "You Know, for Search"
}
```

### Install elasticsearch cloud azure plugin

Stop elasticsearch:

```sh
sudo service elasticsearch stop
```

Install the plugin:

```sh
sudo /usr/share/elasticsearch/bin/plugin -install elasticsearch/elasticsearch-cloud-azure/1.0.0
```

Configure it:

```sh
sudo vi /etc/elasticsearch/elasticsearch.yml
```

And add the following lines:

```yaml
    cloud:
        azure:
            private_key: /home/elasticsearch/keys/azurePrivateKey.key
            certificate: /home/elasticsearch/keys/azureCert.cer
            password: your_password
            subscription_id: your_azure_subscription_id
    discovery:
            type: azure
```

Restart elasticsearch:

```sh
sudo service elasticsearch start
```

### Add new VMs

You just have to add new VMs following the same steps and all your VMs will be able to discuss to each others.

Check your logs. You should see something like:

```
[2013-04-02 15:56:42,891][INFO ][cluster.service          ] [Silver Scorpion] added {[Remnant][of_0GiA3SHCHSUWyIdU5HA][inet[/10.241.192.108:9300]],}, reason: zen-disco-receive(join from node[[Remnant][of_0GiA3SHCHSUWyIdU5HA][inet[/10.241.192.108:9300]]])
```

or

```
[2013-04-02 15:56:42,171][INFO ][cluster.service          ] [Remnant] detected_master [Silver Scorpion][cDXEtIVvRjKgUJWuSaGFRg][inet[/10.241.192.24:9300]], added {[Silver Scorpion][cDXEtIVvRjKgUJWuSaGFRg][inet[/10.241.192.24:9300]],}, reason: zen-disco-receive(from master [[Silver Scorpion][cDXEtIVvRjKgUJWuSaGFRg][inet[/10.241.192.24:9300]]])
```

If something goes wrong, you can add traces using `logging.yml` file:

```yaml
logger:
  # Trace cloud plugin
  cloud: TRACE
```

