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

We will expose here one strategy which to hide our Elasticsearch cluster from outside.

With this strategy, only VM behind this same virtual port can talk to each other. 
That means that with this mode, you can use elasticsearch unicast discovery to build a cluster.

Best, you can use the `elasticsearch-cloud-azure` plugin to let it fetch information about your nodes using
azure API.


Prerequisites
-------------

Before starting, you need to have:

* A [Windows Azure account](http://www.windowsazure.com/)
* SSH keys and certificate

Here is a description on how to generate this using `openssl`:

```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout azure-private.key -out azure-certificate.pem
chmod 600 azure-private.key
openssl x509 -outform der -in azure-certificate.pem -out azure-certificate.cer
```

See this [guide](http://www.windowsazure.com/en-us/manage/linux/how-to-guides/ssh-into-linux/) to have
more details on how to create keys for Azure.

Once done, you need to upload your certificate in Azure:

* Go to the [management console](https://account.windowsazure.com/).
* Sign in using your account.
* Click on `Portal`.
* Go to Settings (bottom of the left list)
* On the bottom bar, click on `Upload` and upload your `azure-certificate.cer` file.


Create Azure VMs
----------------

Connect to [Management Console](https://manage.windowsazure.com/).

Create your first VM by clicking on `+ NEW` button. It will open the following window:

![Create VM assistant](../images/azure/azure-create-vm0.png "Create VM assistant")

Choose `Compute` -> `Virtual Machine` -> `From Gallery`. You will be asked to choose your operating system.

![Create VM assistant](../images/azure/azure-create-vm1.png "Create VM assistant")

Once done, go to next step and choose settings for your VM:

* Virtual machine name
* User name (will be used to log in using SSH later)
* Password (password for this user)
* Size (size of the box: extra small, small...)

Check the `Upload SSH key...` box and upload your PEM Azure certificate (`azure-certificate.cer` or `azure-certificate.pem`).

![Create VM assistant](../images/azure/azure-create-vm2.png "Create VM assistant")

*Note for Mac users, if you don't succeed in clicking on the upload certificate button, just select it
and press the space bar. It will display the upload window...*

Once done, go to next step.

If you are creating the first box, select `Standalone virtual machine` and choose a DNS name for your VM:

![Create VM assistant](../images/azure/azure-create-multivm3.png "Create VM assistant")

If you are creating another box that will join the first one, select `Connect to an existing virtual machine` 
and choose the previous DNS name your entered for the first box:

![Create VM assistant](../images/azure/azure-create-multivm4.png "Create VM assistant")

Once done, go to next step and finally valid your instance creation:

![Create VM assistant](../images/azure/azure-create-vm4.png "Create VM assistant")

### Network settings

Open your VM configuration and click on `Endpoints` tab. Have a look at the SSH Port number. You will need it
to connect to each box in SSH.

If you need to connect to your cluster from *outside*, open 9200 (or 9300) port.


Install elasticsearch
---------------------

You will need to use your azure key from your VM, so you need to copy the private key and the certificate
on your platform, let's say in `/home/elasticsearch/keys` dir.

```sh
# Create a dir for your keys
ssh -i azure-private.key elasticsearch@es-vm2.cloudapp.net 'mkdir /home/elasticsearch/keys'

# Transform private key to PEM format
openssl pkcs8 -topk8 -nocrypt -in azure-private.key -inform PEM -out azure-pk.pem -outform PEM
# Transform certificate to PEM format
openssl x509 -inform der -in azure-certificate.cer -out azure-cert.pem

# Copy your azure private key and certificate
scp -i azure-private.key azure-pk.pem azure-cert.pem elasticsearch@es-vm2.cloudapp.net:/home/elasticsearch/keys
```

Connect with SSH to your VM:

```sh
ssh -i azure-private.key elasticsearch@es-vm2.cloudapp.net
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

Install elasticsearch cloud azure plugin
----------------------------------------

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
            private_key: /home/elasticsearch/keys/azure-pk.pem
            certificate: /home/elasticsearch/keys/azure-cert.pem
            password: your_password
            subscription_id: your_azure_subscription_id
    discovery:
            type: azure
```

Restart elasticsearch:

```sh
sudo service elasticsearch start
```

Add new VMs
-----------

You just have to add new VMs following the same steps and all your VMs will be able to discuss to each others.
Don't forget to choose `Connect to an existing virtual machine` when creating your new VM.

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


Batch it!
Capture une instance (image disk).


