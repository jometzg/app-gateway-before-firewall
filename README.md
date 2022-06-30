# Application Gateway before Firewall - a worked example
There are some circumastances where a workload needs to be protected by both a Web Application Firewall (WAF) and an Azure Firewall. One of the patterns for this is refered to as *application gateweay before firewall* https://docs.microsoft.com/en-us/azure/architecture/example-scenario/gateway/firewall-application-gateway#application-gateway-before-firewall - to quote the documentation:

In this option, inbound web traffic goes through both Azure Firewall and WAF. The WAF provides protection at the web application layer. Azure Firewall acts as a central logging and control point, and it inspects traffic between the Application Gateway and the backend servers. The Application Gateway and Azure Firewall aren't sitting in parallel, but one after the other.

![alt text](images/design4_500.png "Logical diagram")

The rest of this is a worked example of this pattern so you can see what needs to be configured and why.

## The Example Environment

![alt text](images/App-gateway-before-firewall-sample2.png "Logical diagram")

In this sample, we are considering private users needing to access some form of web service on a spoke VNet. This can be any web service hosted in a VM, App Service, Kubernetes etc. But in the case of this exmample, it is a simple Azure Container Instances (ACI) sample app that is hosted in the *aci* subnet of the *spokevnet* VNet. The application gateway is to be configured as listening only on its private IP address and so to test this, there is a Linux VM which is in the same VNet as the application gateway - but a different subnet. The VNets are peered so that requests from the *dmzvnet* VNet can get routed to the ACI-hosted web app in the *spokevnet*.

Without any firewall or network routing configured, the environment looks like this:

![alt text](images/App-gateway-before-firewall-no-routing2.png "Logical diagram")

So, in the test VM
```
curl 10.2.0.4
```
will result in a repsonse from the test app because we have configured application to use the ACI web application as its backend and we are sending the request to the private IP address of the application gateway (10.2.0.4)

## Building the sample - basic environment
The best approach is to build this in stages, verifying that the jump box VM can access the ACI-hosted web application before configuring the routing and the firewall.

### Virtual networks
1. Create the *dmzvnet* - you can use any addresses, but easiest to follow this sample 10.2.0.0/16
2. Add a subnet *gateway* 10.2.0.0/24
3. Add a subnet *AzureFirewallSubnet* 10.2.3.0/24 (the name of this subnet matters)
4. Add a subnet *jump* 10.2.1.0/24 to host the jumpbox
5. Create the *spokevnet* - you can use any addresses, but easiest to follow this sample 10.3.0.0/16
6. Add a subnet *aci* 10.3.2.0/24
7. choosing the dmzvnet, peer this to the spokesubnet

You should now have a pair of VNets with the subnets needed for later. The peering will allow requests from the dmzvnet to go to the spokevnet.
 
### Target ACI-hosted web app
This is a target web app to call. There are any number of ways of doing this, but the fastest is to use Azure Container Instances:
1. Create and AzureContainerInstance
2. Make sue you choose *private* for this
3. Choose a sample web app as its container - aci-helloworld:latest is fine
4. Set its network to be spokevnet and the subnet *aci*

This should result in a web app that is only visible in VNets and will have a private IP address from the range in the aci subnet, for example 10.3.2.4. We can't do much with this, so we need something inside a VNet to call this - hence the next step of provisioning a VM.

Your result ACI overview should look like this:
![alt text](images/aci-settings.png "ACI overview")
Note its private IP address.


### Jump box to test private App Gateway endpoint
The only purpose of this jumpbox VM is to allow us to create requests into the environment that are sourced from some IP within the dmzvnet - this is essentially proxying for some form of internally-facing customer system that needs to access the API. As it's a VM, we can use command-line tools to poke around the environment.

Depending on your security stance, this VM can have a public IP address or be used via Azure Bastion.

To provision:
1. Create a Virtual Machine
2. Choose your favourite distro - I used Ubuntu 20.03
3. For bastion, do not create a public IP address
4. For budget concious, choose a burst VM size
5. In networking, attach this to the dmzvnet using the *jump* subnet

Once this has provisioned, open a shell into the VM:

```
curl 10.3.2.4
```
Should result in the HTML of the container app. If this does not work, check the IP address of your container.


### Application gateway configuration
