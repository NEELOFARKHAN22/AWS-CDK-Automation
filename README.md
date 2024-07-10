# AWS CDK Infrastructure Deployment

## Overview

This project demonstrates how to use AWS CDK to automate the deployment of a basic infrastructure setup. It creates a Virtual Private Cloud (VPC) with one private subnet and one public subnet. Inside the private subnet, an EC2 instance is launched with an inbound rule allowing port 80. The instance's user data starts an NGINX server.

Additionally, a Network Load Balancer (NLB) and a target group are created. The EC2 instance is attached to the target group. A VPC Link is then established for an API Gateway, which uses the NLB as its target. Finally, an API Gateway endpoint is created, allowing access to the NGINX welcome page via an API-invoked URL.

## Features

- Automated creation of AWS infrastructure using AWS CDK.
- Deployment of VPC, EC2 instance with NGINX, Network Load Balancer, and API Gateway.
- Integration of components to demonstrate automated API access to NGINX server.

### AWS Tools and Technologies

| AWS Tool/Technology       | Description |
|-----------------------|-------------|
| AWS CDK               | Infrastructure as Code framework for AWS. |
| API Gateway (REST API) | AWS service for creating, publishing, maintaining, monitoring, and securing REST APIs. |
| Network Load Balancer | AWS service that distributes incoming traffic across multiple targets, such as EC2 instances. |
| Target Groups         | AWS service used with Network Load Balancers or Application Load Balancers to route requests to registered targets. |
| Amazon EC2            | AWS service providing resizable compute capacity in the cloud. |

### Setup Steps

Follow these steps to set up the project:

1. **Install AWS CLI on Ubuntu:**
   ```bash
   $ sudo apt install awscli
   $ sudo apt install curl unzip
   $ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   $ unzip awscliv2.zip
   $ sudo ./aws/install
   ```
### Configure AWS CLI:

Before proceeding with the AWS CDK setup, configure AWS CLI with your credentials and preferred region. Ensure you have your AWS Access Key ID, Secret Access Key, and region (us-east-1) ready. Make sure your IAM user has administrator access. Run the following command and follow the prompts:

```bash
$ aws configure
AWS Access Key ID: YOUR_ACCESS_KEY_ID
AWS Secret Access Key: YOUR_SECRET_ACCESS_KEY
Default region name: us-east-1
Default output format: json
```
### Setup Python and Node.js

#### Install Python 3

```bash
$ sudo apt install python3
$ sudo apt install python3-pip
$ curl -fsSL https://deb.nodesource.com/setup_14.x | sudo -E bash -
$ sudo apt install -y nodejs
```
### setup cdk project in python

```bash
 python3 -m venv env
cdk init app --language python
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pip install aws-cdk.aws-codestar-alpha
```
### added code to neelstact.py
```bash
import aws_cdk as cdk
from aws_cdk import (
    Stack,
    CfnOutput,
    aws_ec2 as ec2,
    aws_elasticloadbalancingv2 as elbv2,
    aws_elasticloadbalancingv2_targets as targets,
    aws_apigateway as apigw,
)
from constructs import Construct
from aws_cdk import App, Environment

class NeelStack(Stack):

    def __init__(self, scope: Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)

        # Create VPC with specified subnets
        vpc = ec2.Vpc(self, "MyVpc",
            max_azs=1,
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    subnet_type=ec2.SubnetType.PUBLIC,
                    cidr_mask=24,
                    name='Public'
                ),
                ec2.SubnetConfiguration(
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS,
                    cidr_mask=24,
                    name='Private'
                )
            ],
        )

        # Define the AMI ID for Ubuntu 20.04 LTS in us-east-1
        ami_id = "ami-0ba8562d785e35387"  # Replace with the actual AMI ID for Ubuntu 20.04 LTS in us-east-1

        # Create a Machine Image object using the AMI ID for us-east-1
        ami = ec2.MachineImage.generic_linux({
            "us-east-1": ami_id
        })

        # Output the AMI ID
        cdk.CfnOutput(self, "AMI_ID", value=ami_id)

        # Create a Security Group
        security_group = ec2.SecurityGroup(self, "MySecurityGroup",
            vpc=vpc,
            description="Allow HTTP inbound traffic",
            allow_all_outbound=True
        )

        # Add an inbound rule to allow HTTP traffic on port 80
        security_group.add_ingress_rule(ec2.Peer.any_ipv4(), ec2.Port.tcp(80), "Allow HTTP traffic")

        # Define the EC2 instance
        instance = ec2.Instance(self, "MyEC2Instance1",
            instance_type=ec2.InstanceType("t2.micro"),
            machine_image=ami,
            vpc=vpc,
            security_group=security_group,
            vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS)
        )

        # User data to install nginx on the instance
        nginx_install_script = """
            # Install Nginx
            sudo apt-get update
            sudo apt-get install nginx -y
            sudo systemctl start nginx
        """
        instance.user_data.add_commands(nginx_install_script)

        # Create Network Load Balancer (NLB)
        nlb = elbv2.NetworkLoadBalancer(self, "MyNLB",
                                        vpc=vpc,
                                        internet_facing=False,
                                        cross_zone_enabled=True
                                        )

        listener_80 = nlb.add_listener("Listener", port=80)

        # Create target group and register instance
        target_group = listener_80.add_targets("TargetGroup",
                                               port=80,
                                               targets=[targets.InstanceTarget(instance)],
                                               health_check=elbv2.HealthCheck(
                                                   path="/",  # Adjust path as per your nginx configuration
                                                   port="80",
                                                   protocol=elbv2.Protocol.HTTP,
                                                   interval=cdk.Duration.seconds(30),  # Check every 30 seconds
                                                   timeout=cdk.Duration.seconds(10),   # Timeout after 10 seconds
                                               )
                                               )

        # Create VPC Link
        vpc_link = apigw.VpcLink(self, "MyVpcLink",
                                 targets=[nlb]
                                 )

        # Create REST API
        api = apigw.RestApi(self, "MyRestApi",
                            endpoint_configuration=apigw.EndpointConfiguration(
                                types=[apigw.EndpointType.REGIONAL]
                            )
                            )

        # Add a resource and method to the API
        resource = api.root.add_resource("mynlb")
        integration = apigw.Integration(
            type=apigw.IntegrationType.HTTP_PROXY,
            integration_http_method="ANY",
            uri=f"http://{nlb.load_balancer_dns_name}",
            options=apigw.IntegrationOptions(
                connection_type=apigw.ConnectionType.VPC_LINK,
                vpc_link=vpc_link
            )
        )
        resource.add_method("ANY", integration)

        # Output the API endpoint URL
        CfnOutput(self, "ApiEndpoint",
                  value=api.url,
                  description="API Gateway Endpoint"
                  )

app = App()
env = Environment(account="851725255821", region="us-east-1")
NeelStack(app, "NeelStack", env=env)
app.synth()
```
#### run commands 
```bash
cdk bootstrap
cdk deploy
```



