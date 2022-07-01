# Two tier K8s application architecture on AWS

## Overview

This guide gives an analysis and architectural view of a 2 Tier Application hosted on AWS.

There are 2 architectural diagrams shared in this guide. The first one gives a multi availability zone layout of an AWS cluster hosting the 2 Tier application, and another is a single availability zone layout of an AWS cluster hosting the 2 Tier application.
The architecture has been planned keeping production use case in mind. However, it does not involve any additional third party security tools as part of it.

This is just a guide and has not been deployed due to limited resources at my end. It should, however, give a clear understanding of the deployment and necessary actions to be taken. It does not contain the detailed step by step procedure to deploy a two tiered application.

## Contents

## Architecture Diagram

<img width="1792" alt="Screenshot 2022-07-01 at 8 09 13 PM" src="https://user-images.githubusercontent.com/53118271/176916374-e3115427-9e6c-4ea5-a8a5-443d49897955.png">

## Architecture Components:

### AWS Infra components:
| AWS Component | Count | Purpose |
|---------------|-------|---------|
| VPC           |   1   | To host entire cluster architecture |
| Private Subnets | 2 | 1 private subnet to host worker nodes & ingress nodes. And another private subnet to host control plane nodes |
| Public Subnets | 1 | To host the NAT Gateway and the internet facing Application Load Balancer |
| Internet Gateway | 1 | To be attached to the VPC to allow traffic communication between the internet and the instances. |
| NAT Gateway | 1 | To allow instances in the private subnets to reach out to the internet via the public subnet and download any packages. |
| Amazon RDS | 3 | To make use of the managed database service of AWS. We should make use of the MySQL engine. To ensure high availability, we can create 2 additional read replicas. |
| Route53 | N/A |   |
| Cloudwatch Metrics | N/A | Tool for monitoring logs, metrics data for audit purposes |
| Cloudformation | N/A |.  |

### EC2 Instance cluster components:
| Instance Name | Instance Type | vCPU | Memory/GB | OS | Purpose |
|-|-|-|-|-|-|
| Master 1 | t3.xlarge | 4 | 16 |   | To host the Kubernetes control plane components. |
| Master 2 | t3.xlarge | 4 | 16 |   | To host the Kubernetes control plane components. This is a replica for high availability. |
| Master 3 | t3.xlarge | 4 | 16 |   | To host the Kubernetes control plane components. This is a replica for high availability. |
| Worker 1 | t3.xlarge | 4 | 16 |   | To host worker plane components and the actual application pod deployments. |
| Worker 2 | t3.xlarge | 4 | 16 |   | To host worker plane components and the actual application pod deployments. |
| Worker 3 | t3.xlarge | 4 | 16 |   | To host worker plane components and the actual application pod deployments. |
| Ingress-node1 | t3.xlarge | 4 | 16 |   | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |
| Ingress-node2 | t3.xlarge | 4 | 16 |   | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |
| Ingress-node3 | t3.xlarge | 4 | 16 |   | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |

### Kubernetes Cluster components:
| Resource Name | Resource Type | Replicas | Purpose |
|-|-|-|-|
| Application-deployment e.g *symbiosis-surveyapp* | Deployment | 6 | Actual application code (tier 1) running as pods. |
| Ingress-gateway-deployment | Deployment | 3 | Ingress gateway pods that simulate a gateway for external traffic into the Kubernetes cluster for accessing application pods. |
| symbiosis-surveyapp-service | Cluster IP Service | N/A | Allows name resolution for the application pods, and technically a single static name/IP to which other pods or services can connect to when needed to access the application pods. This service acts like a virtual IP for the application deployment pods. |
| ingress-gateway-deployment-service | NodePort/LoadBalancer Service | N/A | Allows name resolution for the ingress-gateway pods, and technically a single static name/IP to which other pods or services can connect to when needed to access the ingress-gateway pods. This service acts like a virtual IP for the ingress-gateway pods. |
| Grafana Dashboard | Daemonset | 1 pod per node | For monitoring purposes. This tool helps collect a variety of metrics for Kubernetes clusters. |

## Deployment Summary

1. We first begin with creating the AWS VPC. In this process, we can select 
   1. The VPC cidr network range (e.g 10.0.0.0/16)
   2. Number of availability zones. 3 in our case.
   3. Number of private subnets. 3 in our case. You can configure the subnet ranges of your preference e.g 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
   4. Number of public subnets. 3 in our case. (Note: we can opt for 1 public subnet, put a single NAT Gateway in this single public subnet, and that can work as well. However it will not be a reliable failover mechanism)
   5. Number of NAT Gateways. 3 in our case i.e 1 per availability zone.
   6. Route table object would be automatically populated. This will include public as well as private route tables. These are important to ensure connectivity between subnets in the same availability zone as well as across 3 different availability zones.
   7. An internet gateway also will be created in this process and attached to the VPC.
2. We should create the necessary security groups for communication between the instances and also the Amazon RDS instance we create later.
   1. A security group for all Kubernetes related ports – This should allow connection between the instances only. External traffic should be restricted from accessing Kubernetes ports. Source can be set to the subnet range of the instances.
   2. A security group for allowing Ports 80/443 to Ingress nodes from the source of 0.0.0.0/0.
   3. A security group to allow the Worker nodes to communicate with the MySQL RDS instances on port 3306 (default port for MySQL). Source can be set to the actual IP addresses of the worker nodes.
3. Now that the basic infra components are created, we can go ahead with the creation the EC2 instances that will be running a Kubernetes cluster to host the application. Ensure the above created security groups are added to the appropriate instances
   1. 3 Master nodes – to be put into different availability zones.
   2. 3 worker nodes – to be put into different availability zones.
   3. 3 ingress nodes – to be put into different availability zones.
   4. 1 Bastion/jump host – can be assigned an open security group. This instance will be logged into by the user to perform installation and administration of the Kubernetes cluster. Also, we should be assigning an elastic IP to this instance so that we can ssh into it. ***Kindly note that the architecture diagrams do not contain this bastion/jump host.***
4. Now we need to create the instance target groups that will be having the 3 Ingress nodes as backends to be served by the Application load balancer we create next.
5. Create an internet facing Application (HTTP/HTTPS) load balancer. Its configuration must be to listen on ports 80 (for HTTP) and the backend would be the instance target group we created in the previous step.
6. Now that the infra layer and instances are all set up, we should log into the bastion host, and start off with the installation. In this case, we assume that the upstream Kubernetes deployment is used, however for production, we should make use of vendor supported offerings like RedHat OpenShift, Mirantis Kubernetes Engine, VMware Tanzu, Rancher Labs etc so that we can reach out for any support and expertise needed in case of outage.
7. Once the Kubernetes cluster is deployed, the application can deployed into the Kubernetes cluster. For testing purposes, try to manually deploy a the application and perform curl tests to see if responses are received. Once testing is successful, deployment can be automated as part of pipelines.

## DevOps CI/CD Pipeline

In modern production environments, there is a concept of DevOps CI/CD pipelines. These basically help create an integrate a pipeline (process) for application delivery through various stages like source code repo hosting, code build, packaging (into images), testing, deployment to production. Below is a very basic Pipeline approach that can be implemented through integration with tools like Jenkins/Gitlab:

<img width="1708" alt="Screenshot 2022-07-01 at 8 10 02 PM" src="https://user-images.githubusercontent.com/53118271/176916207-de5cd53a-0da1-4503-b203-3047a49eeba4.png">

It is important to note the different environments present i.e DEV, UAT, PROD . This a very common practice for an application lifecycle. As it goes through the different stages, applications are pushed to different environments. These environments are identical to each other and only differ in scale.


