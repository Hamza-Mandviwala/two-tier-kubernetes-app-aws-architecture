# Two tier K8s application architecture on AWS ##

This GitHub Project gives an analysis and architectural view of a 2 Tier Application hosted on AWS.

There are 2 architectural diagrams shared in this guide. The first one gives a multi availability zone layout of an AWS cluster hosting the 2 Tier application, and another is a single availability zone layout of an AWS cluster hosting the 2 Tier application.
The architecture has been planned keeping production use case in mind. However, it does not involve any additional third party security tools as part of it.

This is just a guide and has not been deployed due to limited resources at my end. It should, however, give a clear understanding of the deployment and necessary actions to be taken. It does not contain the detailed step by step procedure to deploy a two tiered application.

## Contents ##

* [Architecture Diagram](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#architecture-diagram)
* [Architecture Components](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#architecture-components)
  * [AWS Infra components](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#aws-infra-components)
  * [EC2 Instance cluster components](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#ec2-instance-cluster-components)
  * [Kubernetes Cluster components](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#kubernetes-cluster-components)
* [Deployment Summary](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#deployment-summary)
* [DevOps CI/CD Pipeline](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#devops-cicd-pipeline)
* [Bonus](https://github.com/Hamza-Mandviwala/two-tier-k8sapplication-aws-architecture/blob/main/README.md#bonus)

## Architecture Diagram ##

<img width="1535" alt="Screenshot 2022-07-03 at 10 59 41 AM" src="https://user-images.githubusercontent.com/53118271/177026314-b016cbce-5a21-4c0b-a045-977d117366fd.png">

Above is an architecture for a two-tiered k8s application hosted on a production 9 node Kubernetes cluster running on AWS. Tier 1 will be the actual application code that is packaged and running as kubernetes pods on the worker nodes, and Tier 2 would be the database layer i.e Amazon RDS running MySQL in this example.

The red dotted line arrows indicate the flow of application traffic into the cluster, to the application pods and into the database. The blue dotted line arrows indicate users reaching out to Amazon Route53 for address resolution.

When a user out on the internet, tries to access the application e.g by accessing the application url address on their browser, the request is sent to AWS Route53. Here the IP address of the Application load balancer pointing to the cluster is fetched. The user request is then routed over the internet to the application loadbalancer’s IP address via the Internet gateway that is connected to its VPC. The application load balancer load balances the traffic in a round-robin manner to the ‘ingress’ nodes. The ingress nodes basically host the ingress gateway pods that act like a gateway into the Kubernetes cluster for any application data traffic.

The ingress gateway pods have all the necessary routing configurations. Any application traffic hitting these ingress-gateway pods, is then routed to the destination application pods on the respective worker nodes via the help of Kubernetes service that is tied to the application pods. The pods then process the information and store it into the Amazon RDS instances.

## Architecture Components: ##

### AWS Infra components: ###
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

### EC2 Instance cluster components: ###
| Instance Name | Instance Type | vCPU | Memory/GB | OS | Purpose |
|---------------|---------------|------|-----------|----|---------|
| Master 1 | t3.xlarge | 4 | 16 |   | To host the Kubernetes control plane components. |
| Master 2 | t3.xlarge | 4 | 16 |   | To host the Kubernetes control plane components. This is a replica for high availability. |
| Master 3 | t3.xlarge | 4 | 16 |   | To host the Kubernetes control plane components. This is a replica for high availability. |
| Worker 1 | t3.xlarge | 4 | 16 |   | To host worker plane components and the actual application pod deployments. |
| Worker 2 | t3.xlarge | 4 | 16 |   | To host worker plane components and the actual application pod deployments. |
| Worker 3 | t3.xlarge | 4 | 16 |   | To host worker plane components and the actual application pod deployments. |
| Ingress-node1 | t3.xlarge | 4 | 16 |   | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |
| Ingress-node2 | t3.xlarge | 4 | 16 |   | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |
| Ingress-node3 | t3.xlarge | 4 | 16 |   | To host the Kubernetes worker plane components and mainly the ingress gateway pod deployments. |

### Kubernetes Cluster components: ###
| Resource Name | Resource Type | Replicas | Purpose |
|---------------|---------------|----------|---------|
| Application-deployment e.g *symbiosis-surveyapp* | Deployment | 6 | Actual application code (tier 1) running as pods. |
| Ingress-gateway-deployment | Deployment | 3 | Ingress gateway pods that simulate a gateway for external traffic into the Kubernetes cluster for accessing application pods. |
| symbiosis-surveyapp-service | Cluster IP Service | N/A | Allows name resolution for the application pods, and technically a single static name/IP to which other pods or services can connect to when needed to access the application pods. This service acts like a virtual IP for the application deployment pods. |
| ingress-gateway-deployment-service | NodePort/LoadBalancer Service | N/A | Allows name resolution for the ingress-gateway pods, and technically a single static name/IP to which other pods or services can connect to when needed to access the ingress-gateway pods. This service acts like a virtual IP for the ingress-gateway pods. |
| Grafana Dashboard | Daemonset | 1 pod per node | For monitoring purposes. This tool helps collect a variety of metrics for Kubernetes clusters. |

## Deployment Summary ##

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
6. Now that the infra layer and instances are all set up, we should log into the bastion host, and start off with the installation. In this case, we assume that the upstream Kubernetes deployment is used, however for production, we should make use of vendor supported offerings like RedHat OpenShift, Mirantis Kubernetes Engine, VMware Tanzu, Rancher Labs etc so that we can reach out for any support and expertise needed in case of outage. We can also leverage the managed Kubernetes service of AWS known as EKS. This takes away the burden of managing the Kubernetes cluster and we are only responsible for deploying and managing our application workloads.
7. Once the Kubernetes cluster is deployed, the application can deployed into the Kubernetes cluster. For testing purposes, try to manually deploy the application and perform curl tests to see if responses are received. Once testing is successful, deployment can be automated as part of pipelines.

## DevOps CI/CD Pipeline ##

In modern production environments, there is a concept of DevOps CI/CD pipelines. These basically help create an integrate a pipeline (process) for application delivery through various stages like source code repo hosting, code build, packaging (into images), testing, deployment to production. Below is a very basic Pipeline approach that can be implemented through integration with tools like Jenkins/Gitlab:

<img width="1708" alt="Screenshot 2022-07-01 at 8 10 02 PM" src="https://user-images.githubusercontent.com/53118271/176916207-de5cd53a-0da1-4503-b203-3047a49eeba4.png">

It is important to note the different environments present i.e DEV, UAT, PROD . This a very common practice for an application lifecycle. As it goes through the different stages, applications are pushed to different environments. These environments are identical to each other and only differ in scale.

The above image is just one of many example of a CI/CD pipeline. It is important to note that every organization has their own policies, and can come up with their own pipeline that involves different additional stages as well. They can also have additional environments in fact.
1. Initially, a developer would write their code, and push it to a source code repository e.g GitHub, GitLab.
2. For build & packaging, developers/other stake holder would pull the code from the source code repository and build/package them. Typically, this means packaging it into OCI based images, so that it can be run using container runtimes like Docker, Cri-o etc and distributed/deployed using orchestration tools like Kubernetes/Swarm. For this organizations opt for vendor solutions like RedHat OpenShift, Mirantis Kubernetes Engine, VMware Tanzu, Rancher Labs, Amazon EKS, Azure AKS, Google’s GKE etc.
3. Testing is done on the deployed containers/pods at a DEV stage, if it fails, then the code is sent for debugging, and the procedure repeats. If the testing succeeds, it is then pushed to a different environment e.g UAT env for user acceptance testing. The testing can involve multiple different tests depending on the nature of the application e.g UI/UX testing, pen testing, security testing, load testing, acceptance testing, smoke testing etc.
4. Finally, once all tests have passed, and the builds are approved for production, they are put into staging and released for use in the PROD env. This involves deploying the build into production environments that are to host production grade applications. Also, post deployment activities like managing, scaling, troubleshooting, maintaining clusters all come into the Continuous Delivery phase.
Again, the above is not a strict procedure, but just a basic demonstration of a typical CI/CD pipeline. Companies can have their custom pipelines to  meet their needs.

## Bonus ##

<img width="1482" alt="Screenshot 2022-07-03 at 11 15 09 AM" src="https://user-images.githubusercontent.com/53118271/177026673-345098fb-dc84-44a5-a59f-33fbadc99cce.png">

Above is another architecture diagram of a similar 2-tiered application architecture spanning across 3 different availability zones in the same region. 

The red dotted line arrows indicate the flow of application traffic into the cluster, to the application pods and into the database. The blue dotted line arrows indicate users reaching out to Amazon Route53 for address resolution. The solid black line arrows indicate the flow of control in the Kubernetes cluster i.e the master nodes controlling the worker nodes. All Kubernetes administrative traffic happens through this.

Since this cluster spans over 3 availability zones, we need one public subnet in each zone. We also need 2 private subnets (1 to host the worker/ingress nodes, and 1 to host master nodes) in each zone. For the NAT gateways, it is optional to put 1 in each zone, however in this architecture we have shown 1 NAT gateway in each zone to ensure complete failsafe architecture. Important to note that at least 1 NAT gateway is necessary for the traffic to be routed outside the cluster.

The application traffic flow takes a similar route to the one described earlier for the single availability zone cluster. From User client > Internet Gateway > Application Load Balancer > Ingress nodes > Worker nodes > Amazon RDS instance.



