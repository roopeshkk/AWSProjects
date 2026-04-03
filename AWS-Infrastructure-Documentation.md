# AWS Infrastructure Templates Documentation

## Overview

This project contains two AWS CloudFormation templates that provide infrastructure for CI/CD pipelines and networking:

- **`defaultvpc.yaml`** - Recreates a default VPC with public subnets, internet gateway, and security configurations
- **`jenkinec2instance.yaml`** - Deploys a Jenkins CI/CD server on EC2 with necessary permissions and tooling

## Templates

### 1. Default VPC Template (`defaultvpc.yaml`)

#### Purpose
Recreates the default VPC configuration that AWS provides in new accounts, including subnets, internet gateway, route tables, network ACLs, and default security group.

#### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `VPCCIDR` | String | `172.31.0.0/16` | CIDR block for the VPC |

#### Resources Created

##### VPC Configuration
- **VPC**: Main virtual network with DNS support enabled
- **Internet Gateway**: Allows internet access for public subnets
- **VPC Gateway Attachment**: Connects IGW to VPC

##### Networking
- **Public Route Table**: Routes traffic to internet gateway
- **Public Route**: Default route (0.0.0.0/0) via IGW

##### Subnets (3 per region)
- **Public Subnet 1**: `172.31.0.0/20` in first AZ
- **Public Subnet 2**: `172.31.16.0/20` in second AZ
- **Public Subnet 3**: `172.31.32.0/20` in third AZ
- All subnets have `MapPublicIpOnLaunch: true`

##### Security
- **Default Network ACL**: Allows all inbound and outbound traffic
- **Default Security Group**: Allows all outbound traffic and self-referencing inbound rules

#### Outputs

| Output | Description |
|--------|-------------|
| `VPCId` | ID of the created VPC |
| `Subnet1` | ID of first public subnet |
| `Subnet2` | ID of second public subnet |
| `Subnet3` | ID of third public subnet |

#### Usage Example

```bash
# Deploy the VPC
aws cloudformation create-stack \
  --stack-name my-default-vpc \
  --template-body file://defaultvpc.yaml \
  --parameters ParameterKey=VPCCIDR,ParameterValue=10.0.0.0/16

# Get outputs
aws cloudformation describe-stacks --stack-name my-default-vpc --query 'Stacks[0].Outputs'
```

---

### 2. Jenkins EC2 Instance Template (`jenkinec2instance.yaml`)

#### Purpose
Deploys a fully configured Jenkins CI/CD server on Amazon Linux 2 with GitHub integration capabilities, Docker support, and AWS permissions for infrastructure management.

#### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `KeyPairName` | AWS::EC2::KeyPair::KeyName | - | **Required**. Name of existing EC2 key pair for SSH access |
| `AllowedSSHCIDR` | String | `0.0.0.0/0` | CIDR block allowed for SSH access (restrict for security) |

#### Resources Created

##### Security Configuration
- **Jenkins Security Group**: Allows SSH (22), Jenkins web UI (8080), and HTTPS outbound
- **IAM Role**: `JenkinsRole` with comprehensive AWS permissions
- **Instance Profile**: Attaches IAM role to EC2 instance

##### EC2 Instance
- **AMI**: Amazon Linux 2 (`ami-0c02fb55956c7d316`)
- **Instance Type**: `t2.micro` (free tier eligible)
- **User Data**: Automated installation and configuration script

##### IAM Permissions
The Jenkins role includes:
- **AWS Managed Policies**:
  - `CloudWatchAgentServerPolicy`
  - `AmazonSSMManagedInstanceCore`
- **Custom Policy** (`JenkinsCloudFormationPolicy`):
  - CloudFormation full access
  - IAM role management
  - EC2 full access
  - S3 full access
  - CloudWatch Logs access
  - API Gateway access

#### Automated Installation (User Data)

The instance automatically installs and configures:

1. **System Updates**: `yum update -y`
2. **Java 11**: Amazon Corretto (Jenkins requirement)
3. **Git**: Version control system
4. **AWS CLI v2**: AWS command-line interface
5. **Jenkins**: Latest stable version with auto-start
6. **Docker**: Container runtime with Jenkins user permissions
7. **CloudFormation Helper Scripts**: For stack signaling

#### Outputs

| Output | Description |
|--------|-------------|
| `InstanceId` | EC2 instance ID |
| `PublicIP` | Public IP address for access |
| `JenkinsURL` | Direct URL to Jenkins web interface |
| `SSHCommand` | SSH connection command |

#### Usage Example

```bash
# Deploy Jenkins server
aws cloudformation create-stack \
  --stack-name jenkins-server \
  --template-body file://jenkinec2instance.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=my-key-pair \
                ParameterKey=AllowedSSHCIDR,ParameterValue=192.168.1.0/24

# Get Jenkins access information
aws cloudformation describe-stacks --stack-name jenkins-server --query 'Stacks[0].Outputs'
```

#### Post-Deployment Steps

1. **Access Jenkins**: Use the `JenkinsURL` output to access the web interface
2. **Initial Setup**: Complete Jenkins first-time setup wizard
3. **Install Plugins**: Add GitHub, Pipeline, and other required plugins
4. **Configure Credentials**: Set up GitHub credentials for repository access
5. **SSH Access**: Use the `SSHCommand` output for direct server access

#### Security Considerations

- **Restrict SSH CIDR**: Change `AllowedSSHCIDR` from default `0.0.0.0/0`
- **Jenkins Security**: Configure authentication and authorization
- **IAM Permissions**: Review and restrict IAM policy as needed
- **Regular Updates**: Keep Jenkins and system packages updated

## Architecture Overview

```
┌─────────────────┐    ┌──────────────────────┐
│   Default VPC   │    │  Jenkins EC2 Server  │
│                 │    │                      │
│  ┌────────────┐ │    │  ┌─────────────────┐ │
│  │ Public     │ │    │  │   Jenkins       │ │
│  │ Subnets    │ │    │  │   (Port 8080)   │ │
│  │ (3 per AZ) │ │    │  │                 │ │
│  └────────────┘ │    │  └─────────────────┘ │
│                 │    │  ┌─────────────────┐ │
│  ┌────────────┐ │    │  │   Docker        │ │
│  │ Internet   │ │    │  │   Runtime       │ │
│  │ Gateway    │ │    │  │                 │ │
│  └────────────┘ │    │  └─────────────────┘ │
└─────────────────┘    └──────────────────────┘
         │                           │
         └───────────────────────────┘
              CI/CD Pipeline Flow
```

## Deployment Strategy

### Option 1: Deploy VPC First, Then Jenkins
```bash
# 1. Create VPC infrastructure
aws cloudformation create-stack --stack-name my-vpc --template-body file://defaultvpc.yaml

# 2. Deploy Jenkins in the VPC (modify template to use VPC outputs)
aws cloudformation create-stack --stack-name jenkins --template-body file://jenkinec2instance.yaml
```

### Option 2: Standalone Jenkins Deployment
```bash
# Deploy Jenkins in default VPC
aws cloudformation create-stack --stack-name jenkins-standalone --template-body file://jenkinec2instance.yaml
```

## Related Files

- [`defaultvpc.yaml`](defaultvpc.yaml) - VPC infrastructure template
- [`jenkinec2instance.yaml`](jenkinec2instance.yaml) - Jenkins server template

## Maintenance Notes

- **Cost Optimization**: Use t2.micro for development, consider larger instances for production
- **Backup Strategy**: Regularly backup Jenkins configuration and job definitions
- **Monitoring**: Enable CloudWatch monitoring for the EC2 instance
- **Updates**: Keep Jenkins plugins and system packages current
- **Security**: Regularly review and update security groups and IAM permissions</content>
<parameter name="filePath">/Volumes/Data-Master/AWS-Projects/AWSProjects/AWS-Infrastructure-Documentation.md