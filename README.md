######explaination of Main.tf file####


### Provider Configuration
```hcl
provider "aws" {
  region = "ap-south-1"
}
```
- **provider "aws"**: This block specifies that we are using the AWS provider.
- **region = "ap-south-1"**: Sets the region to "ap-south-1" (Mumbai region).

### VPC Creation
```hcl
resource "aws_vpc" "devopsshack_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "devopsshack-vpc"
  }
}
```
- **resource "aws_vpc" "devopsshack_vpc"**: Declares a resource of type "aws_vpc" with the name "devopsshack_vpc".
- **cidr_block = "10.0.0.0/16"**: Sets the CIDR block of the VPC to "10.0.0.0/16".
- **tags**: Tags the VPC with the name "devopsshack-vpc".

### Subnet Creation
```hcl
resource "aws_subnet" "devopsshack_subnet" {
  count = 2
  vpc_id                  = aws_vpc.devopsshack_vpc.id
  cidr_block              = cidrsubnet(aws_vpc.devopsshack_vpc.cidr_block, 8, count.index)
  availability_zone       = element(["ap-south-1a", "ap-south-1b"], count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "devopsshack-subnet-${count.index}"
  }
}
```
- **count = 2**: Creates 2 subnets.
- **vpc_id = aws_vpc.devopsshack_vpc.id**: Associates the subnets with the VPC created earlier.
- **cidr_block = cidrsubnet(aws_vpc.devopsshack_vpc.cidr_block, 8, count.index)**: Calculates the CIDR block for each subnet.
- **availability_zone = element(["ap-south-1a", "ap-south-1b"], count.index)**: Distributes the subnets across two availability zones.
- **map_public_ip_on_launch = true**: Assigns a public IP address to instances launched in the subnet.
- **tags**: Tags each subnet with a unique name.

### Internet Gateway
```hcl
resource "aws_internet_gateway" "devopsshack_igw" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  tags = {
    Name = "devopsshack-igw"
  }
}
```
- **resource "aws_internet_gateway" "devopsshack_igw"**: Declares a resource of type "aws_internet_gateway" with the name "devopsshack_igw".
- **vpc_id = aws_vpc.devopsshack_vpc.id**: Associates the Internet Gateway with the VPC.
- **tags**: Tags the Internet Gateway with the name "devopsshack-igw".

### Route Table
```hcl
resource "aws_route_table" "devopsshack_route_table" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.devopsshack_igw.id
  }

  tags = {
    Name = "devopsshack-route-table"
  }
}
```
- **resource "aws_route_table" "devopsshack_route_table"**: Declares a resource of type "aws_route_table" with the name "devopsshack_route_table".
- **vpc_id = aws_vpc.devopsshack_vpc.id**: Associates the route table with the VPC.
- **route**: Specifies a route within the route table.
  - **cidr_block = "0.0.0.0/0"**: Directs all traffic to the specified gateway.
  - **gateway_id = aws_internet_gateway.devopsshack_igw.id**: Uses the Internet Gateway as the target for the route.
- **tags**: Tags the route table with the name "devopsshack-route-table".

### Route Table Association
```hcl
resource "aws_route_table_association" "a" {
  count          = 2
  subnet_id      = aws_subnet.devopsshack_subnet[count.index].id
  route_table_id = aws_route_table.devopsshack_route_table.id
}
```
- **count = 2**: Associates the route table with the 2 subnets.
- **subnet_id = aws_subnet.devopsshack_subnet[count.index].id**: Associates each subnet with the route table.
- **route_table_id = aws_route_table.devopsshack_route_table.id**: Specifies the route table to associate.

### Security Groups
```hcl
resource "aws_security_group" "devopsshack_cluster_sg" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "devopsshack-cluster-sg"
  }
}
```
- **resource "aws_security_group" "devopsshack_cluster_sg"**: Declares a resource of type "aws_security_group" with the name "devopsshack_cluster_sg".
- **vpc_id = aws_vpc.devopsshack_vpc.id**: Associates the security group with the VPC.
- **egress**: Specifies the outbound traffic rules.
  - **from_port = 0, to_port = 0, protocol = "-1"**: Allows all outgoing traffic.
  - **cidr_blocks = ["0.0.0.0/0"]**: Allows traffic to all destinations.
- **tags**: Tags the security group with the name "devopsshack-cluster-sg".

```hcl
resource "aws_security_group" "devopsshack_node_sg" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "devopsshack-node-sg"
  }
}
```
- **resource "aws_security_group" "devopsshack_node_sg"**: Declares a resource of type "aws_security_group" with the name "devopsshack_node_sg".
- **vpc_id = aws_vpc.devopsshack_vpc.id**: Associates the security group with the VPC.
- **ingress**: Specifies the inbound traffic rules.
  - **from_port = 0, to_port = 0, protocol = "-1"**: Allows all incoming traffic.
  - **cidr_blocks = ["0.0.0.0/0"]**: Allows traffic from all sources.
- **egress**: Specifies the outbound traffic rules.
  - **from_port = 0, to_port = 0, protocol = "-1"**: Allows all outgoing traffic.
  - **cidr_blocks = ["0.0.0.0/0"]**: Allows traffic to all destinations.
- **tags**: Tags the security group with the name "devopsshack-node-sg".

 Absolutely, let's break down each line of this configuration!

### EKS Cluster
```hcl
resource "aws_eks_cluster" "devopsshack" {
  name     = "devopsshack-cluster"
  role_arn = aws_iam_role.devopsshack_cluster_role.arn

  vpc_config {
    subnet_ids         = aws_subnet.devopsshack_subnet[*].id
    security_group_ids = [aws_security_group.devopsshack_cluster_sg.id]
  }
}
```
- **resource "aws_eks_cluster" "devopsshack"**: Declares a resource of type "aws_eks_cluster" with the name "devopsshack".
- **name = "devopsshack-cluster"**: Names the EKS cluster "devopsshack-cluster".
- **role_arn = aws_iam_role.devopsshack_cluster_role.arn**: Specifies the IAM role for the cluster.
- **vpc_config**: Configures the VPC for the cluster.
  - **subnet_ids = aws_subnet.devopsshack_subnet[*].id**: Uses the subnets created earlier.
  - **security_group_ids = [aws_security_group.devopsshack_cluster_sg.id]**: Uses the cluster security group.

### EKS Node Group
```hcl
resource "aws_eks_node_group" "devopsshack" {
  cluster_name    = aws_eks_cluster.devopsshack.name
  node_group_name = "devopsshack-node-group"
  node_role_arn   = aws_iam_role.devopsshack_node_group_role.arn
  subnet_ids      = aws_subnet.devopsshack_subnet[*].id

  scaling_config {
    desired_size = 3
    max_size     = 3
    min_size     = 3
  }

  instance_types = ["t2.large"]

  remote_access {
    ec2_ssh_key = var.ssh_key_name
    source_security_group_ids = [aws_security_group.devopsshack_node_sg.id]
  }
}
```
- **resource "aws_eks_node_group" "devopsshack"**: Declares a resource of type "aws_eks_node_group" with the name "devopsshack".
- **cluster_name = aws_eks_cluster.devopsshack.name**: Associates the node group with the EKS cluster created earlier.
- **node_group_name = "devopsshack-node-group"**: Names the node group "devopsshack-node-group".
- **node_role_arn = aws_iam_role.devopsshack_node_group_role.arn**: Specifies the IAM role for the node group.
- **subnet_ids = aws_subnet.devopsshack_subnet[*].id**: Uses the subnets created earlier.
- **scaling_config**: Configures the scaling behavior of the node group.
  - **desired_size = 3, max_size = 3, min_size = 3**: Sets the desired, maximum, and minimum number of nodes to 3.
- **instance_types = ["t2.large"]**: Specifies the instance type for the nodes.
- **remote_access**: Configures remote access to the nodes.
  - **ec2_ssh_key = var.ssh_key_name**: Specifies the SSH key for remote access.
  - **source_security_group_ids = [aws_security_group.devopsshack_node_sg.id]**: Uses the node security group for remote access.

### IAM Role for EKS Cluster
```hcl
resource "aws_iam_role" "devopsshack_cluster_role" {
  name = "devopsshack-cluster-role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}
```
- **resource "aws_iam_role" "devopsshack_cluster_role"**: Declares a resource of type "aws_iam_role" with the name "devopsshack_cluster_role".
- **name = "devopsshack-cluster-role"**: Names the IAM role "devopsshack-cluster-role".
- **assume_role_policy**: Specifies the trust policy for the role.
  - **"Service": "eks.amazonaws.com"**: Allows the EKS service to assume this role.

### IAM Policy Attachment for EKS Cluster Role
```hcl
resource "aws_iam_role_policy_attachment" "devopsshack_cluster_role_policy" {
  role       = aws_iam_role.devopsshack_cluster_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}
```
- **resource "aws_iam_role_policy_attachment" "devopsshack_cluster_role_policy"**: Declares a resource of type "aws_iam_role_policy_attachment" with the name "devopsshack_cluster_role_policy".
- **role = aws_iam_role.devopsshack_cluster_role.name**: Associates the policy with the IAM role created earlier.
- **policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"**: Specifies the AmazonEKSClusterPolicy policy.

### IAM Role for EKS Node Group
```hcl
resource "aws_iam_role" "devopsshack_node_group_role" {
  name = "devopsshack-node-group-role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}
```
- **resource "aws_iam_role" "devopsshack_node_group_role"**: Declares a resource of type "aws_iam_role" with the name "devopsshack_node_group_role".
- **name = "devopsshack-node-group-role"**: Names the IAM role "devopsshack-node-group-role".
- **assume_role_policy**: Specifies the trust policy for the role.
  - **"Service": "ec2.amazonaws.com"**: Allows the EC2 service to assume this role.

### IAM Policy Attachments for EKS Node Group Role
```hcl
resource "aws_iam_role_policy_attachment" "devopsshack_node_group_role_policy" {
  role       = aws_iam_role.devopsshack_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "devopsshack_node_group_cni_policy" {
  role       = aws_iam_role.devopsshack_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "devopsshack_node_group_registry_policy" {
  role       = aws_iam_role.devopsshack_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}
```
- **resource "aws_iam_role_policy_attachment"**: Declares resources of type "aws_iam_role_policy_attachment".
- **role = aws_iam_role.devopsshack_node_group_role.name**: Associates the policies with the IAM role for the node group.
- **policy_arn**: Specifies the policy ARNs:
  - **"arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"**
  - **"arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"**
  - **"arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"**

      
