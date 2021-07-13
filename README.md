# Deploying OpenShift 4.7 on Equinix Metal
I'm updating the deployment of OpenShift 4.7 on bare metal from Equinix Metal. I did this rather quickly, so your mileage may vary. You should always consider using the official documentation if you are doing something serious! I’m assuming you have:

 - SSH keys configured in Packet
 - A domain registered in AWS Route53 (feel free to use your favorite DNS service)
 - Access to OpenShift subscriptions

I used the Parsippany, USA (EWR1) datacenter, but this should work with any datacenter.

First, deploy the following in EWR1:

 - x1.small.x86 ($0.40/hour)
 - Operating System = Licensed – RHEL 7

This node will act as our “helper”. This is not to be confused with the bootstrap node for deploying OpenShift. We will deploy that later. The “helper” will be where we run the equinixstrap.sh script to get everything ready to go.

Once x1.small.x86 is up and running ssh to it and download the scripts (git isn’t installed by default).

```
# wget https://raw.githubusercontent.com/mperezco/equinixstrap/master/equinixstrap.sh

# wget https://raw.githubusercontent.com/mperezco/equinixstrap/master/fixhaproxy.sh

# wget https://raw.githubusercontent.com/mperezco/equinixstrap/master/imageregistry.sh

# wget https://raw.githubusercontent.com/mperezco/equinixstrap/master/persistentvolumes.sh
```

```
# chmod +x *.sh
```

Now download your pull-secret from the [OpenShift Install Page](https://cloud.redhat.com/openshift/install/pull-secret) and drop it into your current working directory as pull-secret.txt. After that, run the equinixstrap.sh script and pass it three arguments:

 - The pool ID to use that contains the OpenShift subscriptions.
 - The domain name (demonstr8.net below)
 - The sub-domain name and/or cluster name (test below)

```
# ./equinixstrap.sh 8a85f99c6f0fa8e3016f19db8d17768e demonstr8.net test
```

This will take a little bit to run and it does a lot of things. You can [view the script](https://github.com/mperezco/equinixstrap/blob/master/equinixstrap.sh) if you want to see everything it does. In the end, if everything worked you should see this:

```
==== create manifests
INFO Consuming Install Config from target directory
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
INFO Consuming Worker Machines from target directory
INFO Consuming Openshift Manifests from target directory
INFO Consuming OpenShift Install (Manifests) from target directory
INFO Consuming Common Manifests from target directory
INFO Consuming Master Machines from target directory
==== Create publicly accessible directory, Copy ignition files, Create iPXE files
==== all done, you can now iPXE servers to:
http://147.75.199.131:8080/equinixstrap/bootstrap.boot
http://147.75.199.131:8080/equinixstrap/master.boot
http://147.75.199.131:8080/equinixstrap/worker.boot
```


Your IP address will be different of course. As you can see, you are provided with the iPXE boot URLs for the bootstrap, master, and worker nodes. Now you can boot the following in Packet.

 - bootstrap – c2.medium.x86 – custom iPXE – use the bootstrap.boot URL above
 - master0 – c2.medium.x86 – custom iPXE – use the master.boot URL above
 - master1 – c2.medium.x86 – custom iPXE – use the master.boot URL above
 - master2 – c2.medium.x86 – custom iPXE – use the master.boot URL above
 - worker1 – c2.medium.x86 – custom iPXE – use the worker.boot URL above
 - worker2 – c2.medium.x86 – custom iPXE – use the worker.boot URL above

As those boot, you’ll need to get those IP addresses into Amazon Route53 and also change haproxy to have the right IP addresses.

Here are the changes to Route53 I made (as an example)

![DNS Entries in Route53](images/route53.png)

For editing haproxy you can just edit the values in the fixhaproxy.sh and run the script.

```
# vi fixhaproxy.sh
<assign IP addresses>
# ./fixhaproxy.sh
```

Now you can watch and wait to see if the deployment returns

```
# ./openshift-install --dir=packetinstall wait-for bootstrap-complete --log-level=info 
INFO Waiting up to 20m0s for the Kubernetes API at https://api.test.demonstr8.net:6443&#8230;
```

It should look like this if it succeeds

```
# ./openshift-install --dir=packetinstall wait-for bootstrap-complete --log-level=info
INFO Waiting up to 20m0s for the Kubernetes API at https://api.test.demonstr8.net:6443... 
INFO API v1.17.1 up                               
INFO Waiting up to 40m0s for bootstrapping to complete... 
INFO It is now safe to remove the bootstrap resources 
```

Once it returns you can remove the bootstrap server (or comment it out) from /etc/haproxy/haproxy.cfg and restart haproxy.

```
# vi /etc/haproxy/haproxy.cfg
 <comment out bootstrap node>
# systemctl restart haproxy.service
```

Then you can source your kubeconfig and be on your way.

```
# export KUBECONFIG=/root/packetinstall/auth/kubeconfig
# ./oc whoami
```

You can get the nodes and see that the masters are there.

```
# ./oc get nodes
```

The workers will not be there because you need to approve their Certificate Signining Requests (CSR).

```
# ./oc get csr
```

You can approve the pending requests quickly like this.

```
# ./oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs ./oc adm certificate approve
```

Now you should be able to point your browser at the OpenShift console located at https://console-openshift-console.apps.test.demonstr8.net/ where test = cluster name and demonstr8.net = basedomain or $2 and $3 from your equinixstrap.sh command at the start.

If you want to enable an image registry quickly you can do that by running imageregistry.sh. Note that this is not meant for production use as it uses local storage.

```
# ./imageregistry.sh
```

If you want to create some persistent volumes you can run the persistentvolumes.sh script. It will create four persistent volumes on the NFS directory that is exported from the helper node.

```
# ./persistentvolumes.sh
```

Now you can download the [RHEL 8.1 guest image](https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.1/x86_64/product-software), upload it to /var/www/html on the helper node and get to deploying some VMs on [OpenShift Virtualization](https://docs.openshift.com/container-platform/4.4/welcome/index.html)!
