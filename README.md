# Two-tier Kubernetes application architecture on AWS ##

This GitHub Project gives an analysis and architectural view of a 2 Tier Application hosted on AWS.

The architectural diagram shared in this guide focuses on a single availability zone layout of an AWS cluster hosting a 2 Tier application. The architecture has been planned keeping production use case in mind. 

There are 3 tables in this project that give a quick idea of the entire architectural components involved. We can look at it in 3 layers : AWS Infra layer components, EC2 Instance cluster layer components & Kubernetes Cluster layer components.
This architecture does not involve any additional third party security tools as part of it. It simply makes us of the appropriate security groups.

This guide is intended to give a clear understanding of the deployment and necessary actions to be taken. It does not contain the detailed step by step procedure to deploy a two tiered application.

We will also be looking at a typical DevOps CI/CD pipeline that is implemented for an application lifecycle and the stages involved.

Do checkout my [YouTube video](https://youtu.be/_-l50Cct2Uw) explaining the architecture diagram in detail.

P.s Do checkout the final Bonus section to get a tweak of a 2 tiered Kubernetes application architecture on AWS.

## Contents ##

* [Architecture Diagram](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#architecture-diagram)
* [Architecture Components](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#architecture-components)
  * [AWS Infra layer components](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#aws-infra-components)
  * [EC2 Instance Cluster layer components](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#ec2-instance-cluster-components)
  * [Kubernetes Cluster layer components](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#kubernetes-cluster-components)
* [Deployment Summary](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#deployment-summary)
* [DevOps CI/CD Pipeline](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#devops-cicd-pipeline)
* [Bonus](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#bonus)

## Architecture Diagram ##

<img width="1570" alt="Screenshot 2022-07-03 at 3 53 35 PM" src="https://user-images.githubusercontent.com/53118271/177035452-42eef311-4187-49fa-a29a-8a5a42dcff5d.png">

Above is an architecture for a two-tiered k8s application hosted on a production 9 node Kubernetes cluster running on AWS. Tier 1 will be the actual application code that is packaged and running as kubernetes pods on the worker nodes, and Tier 2 would be the database layer i.e Amazon RDS running MySQL in this example.

The red dotted line arrows indicate the flow of application traffic into the cluster, to the application pods and into the database. The blue dotted line arrows indicate users reaching out to Amazon Route53 for address resolution. The solid black line arrows indicate the flow of control in the Kubernetes cluster i.e the master nodes controlling the worker nodes. All Kubernetes administrative traffic happens through this.

When a user out on the internet, tries to access the application e.g by accessing the web application url address on their browser, the request is sent to AWS Route53. Here the IP address of the Application load balancer pointing to the cluster is fetched. The user request is then routed over the internet to the application loadbalancer???s IP address via the Internet gateway that is connected to its VPC. The application load balancer load balances the traffic in a round-robin manner to the ???ingress??? nodes. The ingress nodes basically host the ingress gateway pods that act like a gateway into the Kubernetes cluster for any application data traffic.

The ingress gateway pods have all the necessary routing configurations. Any application traffic hitting these ingress-gateway pods, is then routed to the destination application pods on the respective worker nodes via the help of Kubernetes service that is tied to the application pods. The pods then process the information and store it into the Amazon RDS instances.

## Architecture Components: ##

### AWS Infra layer components: ###
| AWS Component | Count | Purpose |
|---------------|-------|---------|
| VPC           |   1   | To host entire cluster architecture |
| Private Subnets | 2 | 1 private subnet to host worker nodes & ingress nodes. And another private subnet to host control plane master nodes |
| Public Subnets | 1 | To host the NAT Gateway and the internet facing Application Load Balancer |
| Internet Gateway | 1 | To be attached to the VPC to allow traffic communication between the internet and the instances. |
| NAT Gateway | 1 | To allow instances in the private subnets to reach out to the internet via the public subnet and download any packages. |
| Amazon RDS | 3 | To make use of the managed database service of AWS. We should make use of the MySQL engine. To ensure high availability, we can create 2 additional read replicas. |
| Route53 | N/A | To allow public name resolution of the application you wish to deploy. In private clusters where application is only accessed internally, Route53 is not required |
| Cloudwatch Metrics | N/A | An AWS Tool for monitoring logs, metrics data for audit purposes |
| Cloudformation templates | N/A |  An AWS tool used for Infrastructure as Code . Allows easy management and deploying of AWS components. |

### EC2 Instance Cluster layer components: ###
| Instance Name | Instance Type | vCPU | Memory/GB | OS | Purpose |
|---------------|---------------|------|-----------|----|---------|
| Master 1 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host the Kubernetes control plane components. |
| Master 2 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host the Kubernetes control plane components. This is a replica for high availability. |
| Master 3 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host the Kubernetes control plane components. This is a replica for high availability. |
| Worker 1 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host worker plane components and the actual application pod deployments. |
| Worker 2 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host worker plane components and the actual application pod deployments. |
| Worker 3 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host worker plane components and the actual application pod deployments. |
| Ingress-node1 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |
| Ingress-node2 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |
| Ingress-node3 | t3.xlarge | 4 | 16 | Ubuntu/RHEL | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |

### Kubernetes Cluster layer components: ###
| Resource Name | Resource Type | Replicas | Purpose |
|---------------|---------------|----------|---------|
| Application-deployment e.g *symbiosis-webapp* | Deployment | 6 | Actual application code (tier 1) running as pods. |
| Ingress-gateway-deployment | Deployment | 3 | Ingress gateway pods that act as a gateway for external traffic into the Kubernetes cluster for accessing application pods. |
| symbiosis-webapp-service | Cluster IP Service | N/A | Allows name resolution for the application pods, and technically a single static name/IP to which other pods or services can connect to when needed to access the application pods. This service acts like a virtual IP for the application deployment pods. |
| ingress-gateway-deployment-service | NodePort/LoadBalancer Service | N/A | Allows name resolution for the ingress-gateway pods, and technically a single static name/IP to which other pods or services can connect to when needed to access the ingress-gateway pods. This service acts like a virtual IP for the ingress-gateway pods. |
| Grafana Dashboard | Daemonset | 1 pod per node | For monitoring purposes. This tool helps collect a variety of metrics for Kubernetes clusters. |

## Deployment Summary ##

1. We first begin with creating the AWS VPC. In this process, we can select: 
   1. The VPC cidr network range (e.g 10.0.0.0/16)
   2. Number of availability zones. 1 in our case.
   3. Number of private subnets. 2 in our case. You can configure the subnet ranges of your preference e.g 10.0.1.0/24, 10.0.2.0/24.
   4. Number of public subnets. 1 in our case.
   5. Number of NAT Gateways. 1 in our case.
   6. Route table object would be automatically populated. This will include public as well as private route tables. These are important to ensure connectivity between subnets.
   7. An internet gateway also will be created in this process and attached to the VPC.
2. We should create the necessary security groups for communication between the instances and also the Amazon RDS instance we create later.
   1. A security group for all Kubernetes related ports ??? This should allow connection between the instances only. External traffic should be restricted from accessing Kubernetes ports. Source can be set to the subnet range of the instances.
   2. A security group for allowing Ports 80/443 to Ingress nodes from the source of 0.0.0.0/0.
   3. A security group to allow the Worker nodes to communicate with the MySQL RDS instances on port 3306 (default port for MySQL). Source can be set to the actual IP addresses of the worker nodes.
3. Now that the basic infra components are created, we can go ahead with the creation of the EC2 instances that will be running a Kubernetes cluster to host the application. Ensure the above created security groups are added to the appropriate instances.
   1. 3 Master nodes ??? to be put into the master private subnet.
   2. 3 worker nodes ??? to be put into worker plane private subnet.
   3. 3 ingress nodes ??? to be put into worker plane private subnet.
   4. 1 Bastion/jump host ??? can be assigned an open security group. This instance will be logged into by the user to perform installation and administration of the Kubernetes cluster. Also, we should be assigning an elastic IP to this instance so that we can ssh into it. ***Kindly note that the architecture diagrams do not contain this bastion/jump host. You can skip creating a bastion/jump host if you have direct access to the master nodes to spin up a kubernetes cluster.***
4. Now we need to create the instance target groups that will be having the 3 Ingress nodes as backends to be served by the Application load balancer we create next.
5. Create an internet facing Application (HTTP/HTTPS) load balancer. Its configuration must be to listen on ports 80 (for HTTP) and the backend would be the instance target group we created in the previous step.
6. Now that the infra layer and instances are all set up, we should log into the bastion host, and start off with the installation. In this case, we assume that the upstream Kubernetes deployment is used, however for production, we should make use of vendor supported offerings like RedHat OpenShift, Mirantis Kubernetes Engine, VMware Tanzu, Rancher Labs etc so that we can reach out for any support and expertise needed in case of outage. We can also leverage the managed Kubernetes service of AWS known as EKS. This takes away the burden of managing the Kubernetes cluster and we are only responsible for deploying and managing our application workloads.
7. You can now put the monitoring tools in place. For example **Grafana** is a great tool for gathering the relevant metrics for a kubernetes cluster. 
8. Once the Kubernetes cluster is deployed, the application can deployed into the Kubernetes cluster. For testing purposes, try to manually deploy the application and perform curl tests to see if responses are received. Once testing is successful, deployment can be automated as part of pipelines.
9. Once you are satisfied with the results, we can go ahead with creating an Amazon RDS instance with an engine of our choice e.g MySQL.


## DevOps CI/CD Pipeline ##

In modern production environments, there is a concept of DevOps CI/CD pipelines. These basically help create and integrate a pipeline (process) for application delivery through various stages like source code repo hosting, code build, packaging (into images), testing, deployment to production. Below is a very basic Pipeline that can be implemented:

<img width="1708" alt="Screenshot 2022-07-01 at 8 10 02 PM" src="https://user-images.githubusercontent.com/53118271/176916207-de5cd53a-0da1-4503-b203-3047a49eeba4.png">

It is important to note the different environments present i.e DEV, UAT, PROD . This a very common practice for an application lifecycle. As it goes through the different stages, applications are pushed through different environments. These environments are usually identical to each other and only differ in scale.

The above image is just one of many examples of a CI/CD pipeline. It is important to note that every organization has their own policies, and can come up with their own pipeline that involves different additional stages as well. They can also have additional environments in fact.
1. Initially, a developer would write their code, and push it to a source code repository e.g GitHub, GitLab.
2. For build & packaging, developers/other stake holder would pull the code from the source code repository and build/package them. Typically, this means packaging it into OCI based images, so that it can be run using container runtimes like Docker, Cri-o etc and distributed/deployed using orchestration tools like Kubernetes/Swarm. For this, organizations opt for vendor solutions like RedHat OpenShift, Mirantis Kubernetes Engine, VMware Tanzu, Rancher Labs, Amazon EKS, Azure AKS, Google???s GKE etc.
3. Testing is done on the deployed containers/pods at a DEV stage, if it fails, then the code is sent for debugging, and the procedure repeats. If the testing succeeds, it is then pushed to a different environment (*Jenkins can be of help here*) e.g UAT env for user acceptance testing. The testing can involve multiple different tests depending on the nature of the application e.g UI/UX testing, penetration testing, security testing, load testing, acceptance testing, smoke testing etc.
4. Finally, once all tests have passed, and the builds are approved for production, they are put into staging and released for use in the PROD env. This involves deploying the build into production environments that are to host production grade applications. Also, post deployment activities like managing, scaling, troubleshooting, maintaining clusters all come into the Continuous Delivery phase.

Again, the above is not a strict procedure, but just a basic demonstration of a typical CI/CD pipeline. Companies can have their custom pipelines to  meet their needs.

## Criteria check: ##

Below are some criterias this exercise aims to fulfill. Most organizations aim to implement these in their production clusters hosted on the cloud.

| Criteria | Criteria met? | 
|----------|-|
| Cluster in a private isolated network | Yes |
| Web Application accessible over internet | Yes |
| Database tier having restricted access only from the web tier | Yes |
| Managed database and highly available | Yes |

When it comes to managing applications in a production environment, organizations have to be prepared for scalability. There can be 2 solutions for this. One would be to leverage the auto-scaling functionality of AWS. This will help spin up multiple identical EC2 instances on demand whenever required to handle the necessary traffic.

The second (and preferred) way would be to leverage pod deployment scaling functionality. This means that the deployment replicas can be scaled up through the Kubernetes API. This is a much reliable, easier, and cost-effective way of meeting high traffic demands.


## Bonus ##

<img width="1482" alt="Screenshot 2022-07-03 at 11 15 09 AM" src="https://user-images.githubusercontent.com/53118271/177026673-345098fb-dc84-44a5-a59f-33fbadc99cce.png">

Above is another architecture diagram of a similar 2-tiered application architecture spanning across 3 different availability zones in the same region. 

The red dotted line arrows indicate the flow of application traffic into the cluster, to the application pods and into the database. The blue dotted line arrows indicate users reaching out to Amazon Route53 for address resolution. The solid black line arrows indicate the flow of control in the Kubernetes cluster i.e the master nodes controlling the worker nodes. All Kubernetes administrative traffic happens through this.

Since this cluster spans over 3 availability zones, we need one public subnet in each zone. We also need 2 private subnets (1 to host the worker/ingress nodes, and 1 to host master nodes) in each zone. For the NAT gateways, it is optional to put 1 in each zone, however in this architecture we have shown 1 NAT gateway in each zone to ensure complete failsafe architecture. Important to note that at least 1 NAT gateway in the cluster is necessary for the traffic to be routed outside the cluster.

The application traffic flow takes a similar route to the one described earlier for the single availability zone cluster. From User client > Internet Gateway > Application Load Balancer > Ingress nodes > Worker nodes > Amazon RDS instance.

P.s This architecture does not show the pods just because the diagram would get too complex.

## Helpful links:

1. Detailed guide for [RedHat OpenShift 4.10 installation on GCP using UPI (User Provisioned Infrastructure method)](https://github.com/Hamza-Mandviwala/OCP4.10.3-install-GCP-UPI)
2. Deploying a [simple nginxdemo app on minikube](https://github.com/Hamza-Mandviwala/simple-nginx-demo-app-minikube)



