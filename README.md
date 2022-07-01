# Two tier K8s application architecture on AWS

## Overview

This guide gives an analysis and architectural view of a 2 Tier Application hosted on AWS.

There are 2 architectural diagrams shared in this guide. The first one gives a multi availability zone layout of an AWS cluster hosting the 2 Tier application, and another is a single availability zone layout of an AWS cluster hosting the 2 Tier application.
The architecture has been planned keeping production use case in mind. However, it does not involve any additional third party security tools as part of it.

This is just a guide and has not been deployed due to limited resources at my end. It should, however, give a clear understanding of the deployment and necessary actions to be taken.

## Contents

## Deployment Summary

1. We first begin with creating the AWS VPC. In this process, we can select 
a. The VPC cidr network range (e.g 10.0.0.0/16)
b.	Number of availability zones. 3 in our case.
c.	Number of private subnets. 3 in our case. You can configure the subnet ranges of your preference e.g 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
d.	Number of public subnets. 3 in our case. (Note: we can opt for 1 public subnet, put a single NAT Gateway in this single public subnet, and that can work as well. However it will not be a reliable failover mechanism)
e.	Number of NAT Gateways. 3 in our case i.e 1 per availability zone.
f.	Route table object would be automatically populated. This will include public as well as private route tables. These are important to ensure connectivity between subnets in the same availability zone as well as across 3 different availability zones.
g.	An internet gateway also will be created in this process and attached to the VPC.
2.	We should the necessary security groups for communication between the instances and also the Amazon RDS instance we create later.
a.	A security group for all Kubernetes related ports – This should allow connection between the instances only. External traffic should be restricted from accessing Kubernetes ports. Source can be set to the subnet range of the instances.
b.	A security group for allowing Ports 80/443 to Ingress nodes from the source of 0.0.0.0/0.
c.	A security group to allow the Worker nodes to communicate with the MySQL RDS instances on port 3306 (default port for MySQL). Source can be set to the actual IP addresses of the worker nodes.
3.	Now that the basic infra components are created, we can go ahead with the creation the EC2 instances that will be running a Kubernetes cluster to host the application. Ensure the above created security groups are added to the appropriate instances
a.	3 Master nodes – to be put into different availability zones.
b.	3 worker nodes – to be put into different availability zones.
c.	4 ingress nodes – to be put into different availability zones.
d.	1 Bastion/jump host – can be assigned an open security group. This instance will be logged into by the user to perform installation and administration of the Kubernetes cluster. Also, we should be assigning an elastic IP to this instance so that we can ssh into it. Kindly note that the architecture diagrams do not contain this bastion/jump host.
4.	Now we need to create the instance target groups that will be having the 3 Ingress nodes as backends to be served by the Application load balancer we create next.
5.	Create an internet facing Application (HTTP/HTTPS) load balancer. Its configuration must be to listen on ports 80 (for HTTP) and the backend would be the instance target group we created in the previous step.
6.	Now that the infra layer and instances are all set up, we should log into the bastion host, and start off with the installation. In this case, we assume that the upstream Kubernetes deployment is used, however for production, we should make use of vendor supported offerings like RedHat OpenShift, Mirantis Kubernetes Engine, VMware Tanzu, Rancher Labs etc so that we can reach out for any support and expertise needed in case of outage.
7.	Once the Kubernetes cluster is deployed, the application can deployed into the Kubernetes cluster. For testing purposes, try to manually deploy a the application and perform curl tests to see if responses are received. Once testing is successful, deployment can be automated as part of pipelines.

