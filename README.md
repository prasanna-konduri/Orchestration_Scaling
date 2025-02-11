# MERN Stack with Microservices Deployment on AWS

This README provides a comprehensive guide to set up, containerize, and deploy a MERN application with a microservices architecture on AWS.

---

## Table of Contents
1. [Fork the Repository and Pull Updates](#1-fork-the-repository-and-pull-updates)
2. [Set Up the AWS Environment](#2-set-up-the-aws-environment)
3. [Prepare the MERN Application](#3-prepare-the-mern-application)
4. [Version Control with AWS CodeCommit](#4-version-control-with-aws-codecommit)
5. [Continuous Integration with Jenkins](#5-continuous-integration-with-jenkins)
6. [Infrastructure as Code with Boto3](#6-infrastructure-as-code-with-boto3)
7. [Networking and Deployment](#7-networking-and-deployment)
8. [Kubernetes Deployment](#8-kubernetes-deployment)
9. [Monitoring and Logging](#9-monitoring-and-logging)
10. [Conclusion](#10-conclusion)


---

## **1. Fork the Repository and Pull Updates**

### **Steps to Fork**
1. Navigate to the [Sample MERN with Microservices Repository](https://github.com/UnpredictablePrashant/SampleMERNwithMicroservices).
2. Click "Fork" to create a copy under your account.
3. Clone the forked repository:
   ```bash
   git clone https://github.com/<your-username>/SampleMERNwithMicroservices.git
   ```
   
---

## **2. Set Up the AWS Environment**

- Install AWS CLI and configure it with your credentials.
- Install Python and the Boto3 library for programmatically managing AWS resources.

---

## **3. Prepare the MERN Application**

- Create `Dockerfile` for both backend and frontend services to containerize the application.
  ```bash
  # Dockerfile for Frontend

  FROM node:14
  WORKDIR /app
  COPY package*.json ./
  RUN npm install
  COPY . .
  CMD ["npm", "start"]
  EXPOSE 3000

  ```

  ```bash
  # Dockerfile for Backend
  
  FROM node:14
  WORKDIR /app
  COPY package*.json ./
  RUN npm install
  COPY . .
  CMD ["npm", "start"]
  EXPOSE 5000

  ```

- Build Docker images and push them to Amazon ECR.
  
  ```bash
  #Authenticate Docker
  aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com
  ```
![Alt Text](/images/mern-images.JPG)

![Alt Text](/images/mern-ecr-repo.JPG)


---

## **4. Version Control with AWS CodeCommit**

- Create a CodeCommit repository and push your application code to it for version control.
```bash
# Create a CodeCommit repository
aws codecommit create-repository --repository-name MERNApp

# Add the repository as a remote
git remote add codecommit https://git-codecommit.<region>.amazonaws.com/v1/repos/MERNApp

# Push code
git push codecommit main

```

---

## **5. Continuous Integration with Jenkins**

- Install Jenkins on an EC2 instance.
- Create Jenkins pipeline jobs to build Docker images and push them to Amazon ECR.
 



 #### 1.1. Install Jenkins

 1. **Update system and install Java:**

   ```bash
   sudo apt update
   sudo apt install openjdk-11-jdk
   ```
   
 2. **Install Jenkins:**

   ```bash
   wget -q -O - https://pkg.jenkins.io/jenkins.io.key | sudo apt-key add -
   sudo sh -c 'echo deb http://pkg.jenkins.io/debian/ $(lsb_release -cs) main > /etc/apt/sources.list.d/jenkins.list'
   sudo apt update
   sudo apt install jenkins
   ```

 3. **Start Jenkins:**

   ```bash
   sudo systemctl start jenkins
   sudo systemctl enable jenkins
   ```

   ![Alt Text](/images/JK-8.JPG)
   
 4. **Access Jenkins UI:**

   - **Navigate to Jenkins UI:**
     - Open your browser and go to:  
    `http://<your-ec2-public-ip>:8080`

   - **Unlock Jenkins:**
     - Retrieve the initial setup password by running the following command:  
       ```bash
       cat /var/lib/jenkins/secrets/initialAdminPassword
       ```
  - Copy the password and paste it into the Jenkins UI to unlock Jenkins and complete the setup.

   ![Alt Text](/images/JK-1.JPG)

   ![Alt Text](/images/JK-2.JPG)

---

## **6. Infrastructure as Code with Boto3**

- Use Python and Boto3 to define and provision AWS resources like VPC, subnets, security groups, and Auto Scaling Groups programmatically.

  ```bash

  import boto3

  # Initialize the EC2 client
  ec2 = boto3.client('ec2')
  autoscaling = boto3.client('autoscaling')

  # Step 1: Create a VPC
  vpc_response = ec2.create_vpc(CidrBlock='10.0.0.0/16')
  vpc_id = vpc_response['Vpc']['VpcId']
  print(f"VPC Created: {vpc_id}")

  # Tag the VPC
  ec2.create_tags(Resources=[vpc_id], Tags=[{'Key': 'Name', 'Value': 'MERN-VPC'}])

  # Step 2: Create Subnets
  subnet1_response = ec2.create_subnet(
      CidrBlock='10.0.1.0/24',
      VpcId=vpc_id,
      AvailabilityZone='us-east-1a'
  )
  subnet2_response = ec2.create_subnet(
      CidrBlock='10.0.2.0/24',
      VpcId=vpc_id,
      AvailabilityZone='us-east-1b'
  )
  subnet1_id = subnet1_response['Subnet']['SubnetId']
  subnet2_id = subnet2_response['Subnet']['SubnetId']
  print(f"Subnets Created: {subnet1_id}, {subnet2_id}")

  # Step 3: Create an Internet Gateway and Attach to the VPC
  igw_response = ec2.create_internet_gateway()
  igw_id = igw_response['InternetGateway']['InternetGatewayId']
  ec2.attach_internet_gateway(InternetGatewayId=igw_id, VpcId=vpc_id)
  print(f"Internet Gateway Created and Attached: {igw_id}")

  # Step 4: Create a Route Table and Associate with Subnets
  route_table_response = ec2.create_route_table(VpcId=vpc_id)
  route_table_id = route_table_response['RouteTable']['RouteTableId']
  ec2.create_route(
      RouteTableId=route_table_id,
      DestinationCidrBlock='0.0.0.0/0',
      GatewayId=igw_id
  )
  ec2.associate_route_table(RouteTableId=route_table_id, SubnetId=subnet1_id)
  ec2.associate_route_table(RouteTableId=route_table_id, SubnetId=subnet2_id)
  print(f"Route Table Created and Associated: {route_table_id}")

  # Step 5: Create a Security Group
  sg_response = ec2.create_security_group(
      GroupName='MERN-SG',
      Description='Security group for MERN application',
      VpcId=vpc_id
  )
  sg_id = sg_response['GroupId']
  ec2.authorize_security_group_ingress(
      GroupId=sg_id,
      IpPermissions=[
          {
              'IpProtocol': 'tcp',
              'FromPort': 80,
              'ToPort': 80,
              'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
          },
          {
              'IpProtocol': 'tcp',
              'FromPort': 22,
              'ToPort': 22,
              'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
          }
      ]
  )
  print(f"Security Group Created: {sg_id}")

  # Step 6: Create an Auto Scaling Group
  launch_template_response = ec2.create_launch_template(
      LaunchTemplateName='MERN-Launch-Template',
      LaunchTemplateData={
          'ImageId': '<AMI-ID>',
          'InstanceType': 't2.micro',
          'KeyName': '<KEY-PAIR-NAME>',
          'SecurityGroupIds': [sg_id],
          'UserData': 'IyEvYmluL2Jhc2ggCnN1ZG8gYXB0IHVwZGF0ZSAmJiBzdWRvIGFwdCBpbnN0YWxsIGRvY2tlciAtLWEgCnN1ZG8gZG9ja2VyIHJ1bg=='  # Base64 encoded startup script
      }
  )
  launch_template_id = launch_template_response['LaunchTemplate']['LaunchTemplateId']

  asg_response = autoscaling.create_auto_scaling_group(
      AutoScalingGroupName='MERN-ASG',
      LaunchTemplate={
          'LaunchTemplateId': launch_template_id,
          'Version': '$Latest'
      },
      MinSize=1,
      MaxSize=3,
      DesiredCapacity=2,
      VPCZoneIdentifier=f"{subnet1_id},{subnet2_id}"
  )
  print(f"Auto Scaling Group Created: MERN-ASG")


 - Replace <AMI-ID> with the AMI ID of your Dockerized backend/frontend.
 - Replace <KEY-PAIR-NAME> with your EC2 key pair name.

 
---

## **7. Networking and Deployment**

- Set up an Elastic Load Balancer (ELB) for traffic distribution.
- Configure Route 53 for DNS routing.
- Deploy the frontend and backend services on EC2 instances.

---

## **8. Kubernetes Deployment**

### **Steps to Achieve Kubernetes Deployment:**

1. **Install `eksctl`:**
   - Ensure `eksctl` is installed on your local machine. Use the following command to install it:
     ```bash
     curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_$(uname -m).tar.gz" | tar xz -C /tmp
     sudo mv /tmp/eksctl /usr/local/bin
     ```

2. **Create an Amazon EKS Cluster:**
   - Use `eksctl` to create the Kubernetes cluster:
     ```bash
     eksctl create cluster \
       --name mern-cluster \
       --region us-west-2 \
       --nodegroup-name standard-workers \
       --node-type t3.medium \
       --nodes 3 \
       --nodes-min 1 \
       --nodes-max 4 \
       --managed
     ```

3. **Install Helm:**
   - Helm is required for packaging and deploying the MERN application. Install Helm with:
     ```bash
     curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
     ```

     ![Alt Text](/images/cap-helm-installation.JPG)

4. **Create Kubernetes Manifests:**
   - Write Helm charts or YAML manifests for your MERN application components (frontend, backend). Example for the backend:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: backend-deployment
     spec:
       replicas: 2
       selector:
         matchLabels:
           app: backend
       template:
         metadata:
           labels:
             app: backend
         spec:
           containers:
           - name: backend
             image: <ECR-backend-image-url>
             ports:
             - containerPort: 5000
     ```

5. **Deploy Using Helm:**
   - Package the Helm chart and deploy it:
     ```bash
     helm install mern-backend ./backend-chart
     helm install mern-frontend ./frontend-chart
     ```

6. **Expose Services:**
   - Use `LoadBalancer` services to expose your frontend and backend applications:
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: backend-service
     spec:
       type: LoadBalancer
       selector:
         app: backend
       ports:
       - protocol: TCP
         port: 80
         targetPort: 5000
     ```

---

## **9. Monitoring and Logging**

### **Steps to Achieve Monitoring and Logging:**

1. **Enable CloudWatch Monitoring:**
   - Ensure CloudWatch monitoring is enabled for your EKS cluster. Add the necessary IAM permissions for CloudWatch to your EKS worker nodes:
     ```json
     {
       "Effect": "Allow",
       "Action": [
         "cloudwatch:*",
         "logs:*"
       ],
       "Resource": "*"
     }
     ```

2. **Install `kubectl` Metrics Server:**
   - Deploy Kubernetes Metrics Server to monitor pod resource usage:
     ```bash
     kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
     ```

3. **Set Up Alarms in CloudWatch:**
   - Use CloudWatch to set up alarms for CPU or memory usage thresholds. Example steps:
     - Go to the **CloudWatch** console.
     - Create an alarm on a metric (e.g., CPU utilization).
     - Configure SNS notifications for alarms.

4. **Enable Logging:**
   - Ensure logs from pods are collected in CloudWatch Logs:
     ```bash
     aws eks update-cluster-config \
       --region us-west-2 \
       --name mern-cluster \
       --logging '{"clusterLogging":[{"types":["api","audit","authenticator"],"enabled":true}]}'
     ```

5. **Centralize Logs with Fluentd:**
   - Deploy Fluentd to send logs from your EKS cluster to CloudWatch:
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-cloudwatch.yaml
     ```

## **10. Conclusion**

This project demonstrates the end-to-end deployment of a scalable MERN stack application with microservices architecture on AWS. By leveraging modern cloud-native technologies such as Docker, Kubernetes, and AWS services like EKS, CloudWatch, and Route 53, the application achieves:

- **Scalability:** Kubernetes ensures the application can handle increasing loads by scaling pods dynamically.
- **High Availability:** AWS infrastructure, combined with Elastic Load Balancers and managed Kubernetes clusters, provides resilience and ensures minimal downtime.
- **Automation:** Tools like Jenkins and Helm streamline the CI/CD process, enabling rapid deployment and updates.
- **Monitoring and Logging:** CloudWatch integration provides centralized monitoring and logging, ensuring quick identification and resolution of issues.

Through this deployment, we have created a robust, secure, and efficient cloud-based solution, showcasing the power of containerized microservices and AWS for modern application development. This architecture can be further extended with additional microservices, serverless components, or advanced security features to meet evolving requirements.

The comprehensive documentation and modular design also allow for easy reproduction of this setup, enabling developers to adapt the process for similar applications or projects.


## **11. Contributing**

I welcome contributions! To contribute:

1. Fork the repository.
2. Create a new branch for your feature or bug fix.
3. Commit your changes with clear messages.
4. Submit a pull request for review.

Make sure to follow the code style guidelines and include proper documentation for any new features.


## **12. Contact**

For any queries, feel free to contact me:

- **Email:** adityavakharia@gmail.com
- **GitHub:** [Aditya-rgb](https://github.com/Aditya-rgb/Deploying-and-Scaling-Web-Application)

You can also open an issue in the repository for questions or suggestions.




