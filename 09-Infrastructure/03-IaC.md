# Infrastructure as Code

## Title & Summary

**Infrastructure as Code (IaC)** is the practice of managing and provisioning infrastructure through machine-readable definition files rather than physical hardware configuration or interactive tools. This pattern covers declarative vs imperative approaches, tools like Terraform and CloudFormation, state management, and best practices for treating infrastructure like software.

---

## Problem Statement

Traditional infrastructure management faces significant challenges:

- **Manual configuration** is error-prone and inconsistent
- **Documentation drift** - infrastructure changes aren't tracked
- **Reproducibility** - difficult to recreate environments
- **Slow provisioning** - hours/days to set up new environments
- **Configuration drift** - environments diverge over time
- **Lack of version control** - no history of infrastructure changes

### Example Scenario

```
Traditional Infrastructure Management:

┌─────────────────────────────────────────────────┐
│  Manual Process:                                │
│  1. Login to cloud console                      │
│  2. Click through UI to create resources        │
│  3. Manually configure networking               │
│  4. SSH into servers to install software        │
│  5. Document changes (if remembered)            │
│  6. Repeat for each environment                 │
└─────────────────────────────────────────────────┘

Problems:
├── Different environments have different configs
├── No audit trail of changes
├── Takes hours to provision new environment
├── Difficult to rollback changes
└── Knowledge trapped in individuals' heads
```

---

## Solution

### Declarative vs Imperative Approaches

```
Imperative (Procedural):
┌─────────────────────────────────────────────────┐
│  "HOW to do it" - Step-by-step instructions     │
│                                                  │
│  1. Create VPC with CIDR 10.0.0.0/16            │
│  2. Create subnet 10.0.1.0/24 in AZ-1            │
│  3. Create subnet 10.0.2.0/24 in AZ-2            │
│  4. Create internet gateway                      │
│  5. Attach gateway to VPC                        │
│  6. Create route table                          │
│  7. Add route 0.0.0.0/0 → igw                   │
└─────────────────────────────────────────────────┘

Declarative:
┌─────────────────────────────────────────────────┐
│  "WHAT you want" - Desired state definition     │
│                                                  │
│  resource "aws_vpc" "main" {                    │
│    cidr_block = "10.0.0.0/16"                   │
│    enable_dns_hostnames = true                  │
│  }                                              │
│                                                  │
│  (Tool figures out HOW to achieve this state)   │
└─────────────────────────────────────────────────┘
```

### Terraform - Declarative Infrastructure

```hcl
# terraform/main.tf

terraform {
  required_version = ">= 1.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

# Variables for flexibility
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project_name" {
  description = "Project name for tagging"
  type        = string
}

# VPC Module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.0.0"
  
  name = "${var.project_name}-${var.environment}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway   = true
  single_nat_gateway   = false
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project_name}-${var.environment}"
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.0.0"
  
  cluster_name    = "${var.project_name}-${var.environment}-eks"
  cluster_version = "1.28"
  
  cluster_endpoint_public_access  = true
  cluster_certificate_authority   = true
  
  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.private_subnets
  
  eks_managed_node_groups = {
    general = {
      min_size     = 2
      max_size     = 10
      desired_size = 3
      
      instance_types = ["t3.medium", "t3.large"]
      
      tags = {
        Name = "general-nodes"
      }
    }
    
    gpu = {
      min_size     = 0
      max_size     = 2
      desired_size = 0
      
      instance_types = ["g4dn.xlarge"]
      
      tags = {
        Name = "gpu-nodes"
      }
    }
  }
  
  tags = {
    Environment = var.environment
  }
}

# Outputs
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
}

output "cluster_certificate_authority_data" {
  description = "EKS cluster certificate authority data"
  value       = module.eks.cluster_certificate_authority_data
  sensitive   = true
}
```

### Terraform Modules - Reusable Components

```hcl
# modules/web-server/main.tf

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 2
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "ami_id" {
  description = "AMI ID"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "subnet_ids" {
  description = "Subnet IDs"
  type        = list(string)
}

variable "security_group_ids" {
  description = "Security group IDs"
  type        = list(string)
}

resource "aws_launch_template" "web" {
  name_prefix   = "web-server-"
  description   = "Launch template for web servers"
  image_id      = var.ami_id
  instance_type = var.instance_type
  
  vpc_security_group_ids = var.security_group_ids
  
  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    app_port = 80
  }))
  
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "web-server"
      Type = "web"
    }
  }
}

resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
  
  vpc_zone_identifier = var.subnet_ids
  min_size            = var.instance_count
  max_size            = var.instance_count * 3
  desired_capacity    = var.instance_count
  
  health_check_type     = "ELB"
  health_check_grace_period = 300
  
  tag {
    key                 = "Name"
    value               = "web-server"
    propagate_at_launch = true
  }
}

output "asg_arn" {
  value = aws_autoscaling_group.web.arn
}
```

### AWS CloudFormation - Native AWS IaC

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Serverless API with API Gateway, Lambda, and DynamoDB'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod

Resources:
  ApiTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub '${Environment}-api-table'
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: PK
          AttributeType: S
        - AttributeName: SK
          AttributeType: S
      KeySchema:
        - AttributeName: PK
          KeyType: HASH
        - AttributeName: SK
          KeyType: RANGE
      GlobalSecondaryIndexes:
        - IndexName: GSI-ByUserId
          KeySchema:
            - AttributeName: UserId
              KeyType: HASH
            - AttributeName: SK
              KeyType: RANGE
          Projection:
            ProjectionType: ALL
          ProjectionType: ALL
      Tags:
        - Key: Environment
          Value: !Ref Environment

  ApiRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: DynamoDBAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                  - dynamodb:UpdateItem
                  - dynamodb:DeleteItem
                Resource: !GetAtt ApiTable.Arn

  ApiFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub '${Environment}-api-function'
      Runtime: nodejs18.x
      Handler: index.handler
      Role: !GetAtt ApiRole.Arn
      Code:
        S3Bucket: !Sub '${Environment}-deployments'
        S3Key: api-function.zip
      MemorySize: 256
      Timeout: 30
      Environment:
        Variables:
          TABLE_NAME: !Ref ApiTable
          ENVIRONMENT: !Ref Environment
      Tags:
        Environment: !Ref Environment

  ApiGateway:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: !Sub '${Environment}-api'
      Description: API Gateway for serverless API

  ApiMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ApiGateway
      ResourceId: !GetAtt ApiGateway.RootResourceId
      HttpMethod: ANY
      AuthorizationType: NONE
      AuthorizerId: !Ref ApiAuthorizer
      RequestModels:
        application/json: String
      Integration:
        Type: AWS
        HttpMethod: POST
        Uri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${ApiFunction.Arn}/invocations'
        RequestTemplates:
          application/json: !Sub |
            {
              "statusCode": $input.json('$.statusCode'),
              "body": $util.base64Encode($input.body)
            }
        PassthroughBehavior: WHEN_NO_MATCH
        IntegrationResponses:
          - StatusCode: 200
            SelectionPattern: 200
            ResponseParameters:
              method.response.header.Access-Control-Allow-Origin: "'*'"
              method.response.header.Access-Control-Allow-Headers: "'Content-Type,Authorization'"
              method.response.header.Access-Control-Allow-Methods: "'GET,OPTIONS,PUT,POST,DELETE'"
            ResponseTemplates:
              application/json: !Sub |
                #set($inputRoot = $input.path('$'))
                {
                  "statusCode": $inputRoot.statusCode,
                  "body": $util.base64Decode($inputRoot.body)
                }
      MethodResponses:
        - StatusCode: 200
          ResponseParameters:
            method.response.header.Access-Control-Allow-Origin: true
            method.response.header.Access-Control-Allow-Headers: true
            method.response.header.Access-Control-Allow-Methods: true
          ResponseModels:
            application/json: String

  ApiDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn: ApiMethod
    Properties:
      RestApiId: !Ref ApiGateway
      StageName: live

Outputs:
  ApiEndpoint:
    Description: API Gateway endpoint URL
    Value: !Sub 'https://${ApiGateway}.execute-api.${AWS::Region}.amazonaws.com/live'
  TableArn:
    Description: DynamoDB table ARN
    Value: !GetAtt ApiTable.Arn
```

### Ansible - Configuration Management

```yaml
# ansible/playbook.yml

---
- name: Deploy Web Application
  hosts: webservers
  become: yes
  vars:
    app_name: myapp
    app_port: 8080
    app_user: appuser
    java_version: "11"
    
  tasks:
    # Install Java
    - name: Install OpenJDK
      ansible.builtin.apt:
        name: openjdk-{{ java_version }}-jdk
        state: present
        update_cache: yes
    
    # Create application user
    - name: Create application user
      ansible.builtin.user:
        name: "{{ app_user }}"
        shell: /bin/bash
        create_home: yes
        groups: sudo
    
    # Create application directory
    - name: Create application directory
      ansible.builtin.file:
        path: "/opt/{{ app_name }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'
    
    # Deploy application
    - name: Copy application files
      ansible.builtin.copy:
        src: ./app/
        dest: "/opt/{{ app_name }}/"
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0644'
    
    # Configure systemd service
    - name: Create systemd service file
      ansible.builtin.template:
        src: templates/app.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        owner: root
        group: root
        mode: '0644'
    
    # Enable and start service
    - name: Enable application service
      ansible.builtin.systemd:
        name: "{{ app_name }}"
        enabled: yes
        state: started
        daemon_reload: yes
    
    # Configure firewall
    - name: Allow application port
      ansible.builtin.ufw:
        port: "{{ app_port }}"
        proto: tcp
        action: allow
    
    # Setup monitoring
    - name: Install monitoring agent
      ansible.builtin.package:
        name: prometheus-node-exporter
        state: present
```

---

## When to Use

### Use Terraform When:
- ✅ Multi-cloud infrastructure
- ✅ Need state management
- ✅ Want declarative approach
- ✅ Building complex infrastructure

### Use CloudFormation When:
- ✅ AWS-only infrastructure
- ✅ Need native AWS integration
- ✅ Using AWS-specific features

### Use Ansible When:
- ✅ Configuration management
- ✅ Application deployment
- ✅ Need agentless approach
- ✅ Post-provisioning tasks

---

## Tradeoffs

| Tool | Multi-Cloud | Learning Curve | State Management | Best For |
|------|-------------|----------------|------------------|----------|
| **Terraform** | ✅ Yes | Medium | ✅ Built-in | Infrastructure provisioning |
| **CloudFormation** | ❌ AWS only | Medium | ✅ Built-in | AWS-native infrastructure |
| **Ansible** | ✅ Yes | Low | ❌ Stateless | Configuration management |
| **Pulumi** | ✅ Yes | Medium | ✅ Built-in | Infrastructure as real code |

### Quantitative Comparison

```
Provisioning Time Comparison:

Manual Infrastructure:
├── VPC + Subnets: 15 minutes
├── Security Groups: 10 minutes
├── EC2 Instances: 10 minutes
├── Load Balancer: 10 minutes
├── RDS Database: 20 minutes
├── Configuration: 30 minutes
└── Total: ~95 minutes

Terraform:
├── Write configuration: 30 minutes (one-time)
├── terraform apply: 15 minutes
└── Total: 15 minutes (after initial setup)

CloudFormation:
├── Write template: 30 minutes (one-time)
├── Stack creation: 12 minutes
└── Total: 12 minutes (after initial setup)

Cost Savings:
- Reduced provisioning time: 80% faster
- Fewer errors: 90% reduction
- Consistent environments: 100% reproducible
```

---

## Implementation Example

### Complete Terraform Project Structure

```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── outputs.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── outputs.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── outputs.tf
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   └── rds/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
├── scripts/
│   ├── bootstrap.sh
│   └── validate.sh
├── .terraform.lock.hcl
├── .gitignore
└── README.md
```

### Environment-Specific Configuration

```hcl
# environments/prod/main.tf

terraform {
  backend "s3" {
    bucket         = "mycompany-prod-terraform-state"
    key            = "infrastructure/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks-prod"
  }
}

module "production_vpc" {
  source = "../../modules/vpc"
  
  environment = "production"
  cidr        = "10.0.0.0/16"
  
  azs = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  private_subnets = [
    "10.0.1.0/24",
    "10.0.2.0/24",
    "10.0.3.0/24"
  ]
  
  public_subnets = [
    "10.0.101.0/24",
    "10.0.102.0/24",
    "10.0.103.0/24"
  ]
  
  enable_nat_gateway = true
  
  tags = {
    Environment = "production"
    CostCenter  = "engineering"
    Owner       = "platform-team"
  }
}

module "production_eks" {
  source = "../../modules/eks"
  
  environment = "production"
  
  vpc_id     = module.production_vpc.vpc_id
  subnet_ids = module.production_vpc.private_subnets
  
  cluster_version = "1.28"
  
  node_groups = {
    general = {
      min_size     = 3
      max_size     = 20
      desired_size = 5
      instance_types = ["m5.large", "m5.xlarge"]
    }
  }
  
  encryption_config = {
    resources = ["secrets"]
  }
  
  tags = {
    Environment = "production"
  }
}
```

---

## Anti-Pattern

### ❌ Anti-Pattern: Hardcoded Values

```hcl
# Bad: Hardcoded values
resource "aws_instance" "web" {
  ami           = "ami-12345678"  # Hardcoded!
  instance_type = "t2.micro"      # Hardcoded!
  
  tags = {
    Name = "web-server"  # No environment context
  }
}
```

### ❌ Anti-Pattern: Storing Secrets in Code

```hcl
# Bad: Secrets in code
resource "aws_db_instance" "main" {
  username = "admin"
  password = "supersecretpassword123"  # NEVER DO THIS!
}
```

### ❌ Anti-Pattern: No State Locking

```hcl
# Bad: No state locking configured
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "terraform.tfstate"
    # Missing dynamodb_table for locking!
  }
}
```

### ✅ Correct Approach

```hcl
# Good: Use variables and secrets management
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "environment" {
  type = string
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  
  tags = {
    Name        = "${var.environment}-web-server"
    Environment = var.environment
  }
}

# Use AWS Secrets Manager or Parameter Store
resource "aws_db_instance" "main" {
  username = "admin"
  password = aws_secretsmanager_secret_version.db_password.secret_string
}

# Proper state configuration with locking
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # State locking!
  }
}
```

---

## Related Patterns

- **[Containerization](01-Containerization.md)** - Container-based infrastructure
- **[Orchestration](02-Orchestration.md)** - Kubernetes as code
- **[CI/CD](../11-DevOps/01-CI-CD.md)** - Infrastructure pipelines
- **[Security Patterns](../05-Safety-Engineering/01-Security-Patterns.md)** - Secure infrastructure
- **[Observability](../10-Observability/01-Logging.md)** - Infrastructure monitoring