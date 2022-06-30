# Application Gateway before Firewall - a worked example
There are some circumastances where a workload needs to be protected by both a Web Application Firewall (WAF) and an Azure Firewall. One of the patterns for this is refered to as *application gateweay before firewall* https://docs.microsoft.com/en-us/azure/architecture/example-scenario/gateway/firewall-application-gateway#application-gateway-before-firewall - to quote the documentation:

In this option, inbound web traffic goes through both Azure Firewall and WAF. The WAF provides protection at the web application layer. Azure Firewall acts as a central logging and control point, and it inspects traffic between the Application Gateway and the backend servers. The Application Gateway and Azure Firewall aren't sitting in parallel, but one after the other.

![alt text](images/design4_500.png "Logical diagram")

The rest of this is a worked example of this pattern so you can see what needs to be configured and why.

## The Example Environment

![alt text](images/App-gateway-before-firewall-sample.png "Logical diagram")
