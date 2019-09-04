## Dockerの基本的な操作を試す

### クラスタの構築
rke-env.yamlを用いてCloudFormationにて実行し、スタックを作成

+ Managerサーバ
+ ControlePlane
+ Workerノード1
+ Workerノード2
+ その他ネットワーク関連、IAM等のリソース

が作成される

```yaml
###############################################
# 概要
# Dockerをインストール済みのEC2とVPC及びSSMアクセスするためのロール等を作成します
# 東京リージョンを想定しています
# RKEクラスタ構築用。See:https://rancher.com/blog/2018/2018-05-14-rke-on-aws/
###############################################

AWSTemplateFormatVersion: '2010-09-09'

###############################################
# Parameters
###############################################
Parameters:
  ProjectName:
    Type: String
    Default: KubernetesDemo

  # EC2インスタンス用のキーペアはEC2マネジメントコンソールで事前に作成し、キーペア名を入力してください
  EC2KeyPair:
    Type: AWS::EC2::KeyPair::KeyName

###############################################
# Resources
###############################################
Resources:
  Vpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 172.16.0.0/25
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
      InstanceTenancy: default
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-vpc

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-igw

  IgwAttachVpc:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref Vpc

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      AvailabilityZone: 'ap-northeast-1a'
      CidrBlock: 172.16.0.0/27
      MapPublicIpOnLaunch: 'true'
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-public-subnet

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref Vpc
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-public-rtb
        - Key: kubernetes.io/cluster/2019
          Value: owned

  RouteAddInternet:
    Type: AWS::EC2::Route
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
      RouteTableId: !Ref PublicRouteTable

  AssociatePublicSubnet1ToPublicRouteTable:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet

  Ec2Securitygroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub ${ProjectName}-ec2-sg
      GroupDescription: !Sub ${ProjectName}-ec2-sg
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-ec2-sg
        - Key: kubernetes.io/cluster/2019
          Value: owned
      VpcId: !Ref Vpc
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: '80'
        ToPort: '80'
        CidrIp: '0.0.0.0/0'
      - IpProtocol: -1
        FromPort: -1
        ToPort: -1
        CidrIp: '172.16.0.0/27'

  ControlePlane:
    Type: AWS::EC2::Instance
    Properties:
      AvailabilityZone: ap-northeast-1a
      ImageId: ami-0d7ed3ddb85b521a6
      InstanceType: t2.large
      KeyName: !Ref EC2KeyPair
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            VolumeSize: 20
      SecurityGroupIds:
        - !Ref Ec2Securitygroup
      SubnetId: !Ref PublicSubnet
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-ControlePlane
        - Key: kubernetes.io/cluster/2019
          Value: owned
      IamInstanceProfile: !Ref Ec2IAMInstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y docker
          systemctl enable docker
          service docker start
          sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
          sudo chmod +x /usr/local/bin/docker-compose
          sudo usermod -a -G docker ec2-user

  Worker1:
    Type: AWS::EC2::Instance
    Properties:
      AvailabilityZone: ap-northeast-1a
      ImageId: ami-0d7ed3ddb85b521a6
      InstanceType: t2.large
      KeyName: !Ref EC2KeyPair
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            VolumeSize: 20
      SecurityGroupIds:
        - !Ref Ec2Securitygroup
      SubnetId: !Ref PublicSubnet
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-Worker1
        - Key: kubernetes.io/cluster/2019
          Value: owned
      IamInstanceProfile: !Ref Ec2IAMInstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y docker
          systemctl enable docker
          service docker start
          sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
          sudo chmod +x /usr/local/bin/docker-compose
          sudo usermod -a -G docker ec2-user

  Worker2:
    Type: AWS::EC2::Instance
    Properties:
      AvailabilityZone: ap-northeast-1a
      ImageId: ami-0d7ed3ddb85b521a6
      InstanceType: t2.large
      KeyName: !Ref EC2KeyPair
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            VolumeSize: 20
      SecurityGroupIds:
        - !Ref Ec2Securitygroup
      SubnetId: !Ref PublicSubnet
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-Worker2
        - Key: kubernetes.io/cluster/2019
          Value: owned
      IamInstanceProfile: !Ref Ec2IAMInstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y docker
          systemctl enable docker
          service docker start
          sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
          sudo chmod +x /usr/local/bin/docker-compose
          sudo usermod -a -G docker ec2-user

  Manager:
    Type: AWS::EC2::Instance
    Properties:
      AvailabilityZone: ap-northeast-1a
      ImageId: ami-0d7ed3ddb85b521a6
      InstanceType: t2.large
      KeyName: !Ref EC2KeyPair
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            VolumeSize: 10
      SecurityGroupIds:
        - !Ref Ec2Securitygroup
      SubnetId: !Ref PublicSubnet
      Tags:
        - Key: Name
          Value: !Sub ${ProjectName}-Manager
        - Key: kubernetes.io/cluster/2019
          Value: owned
      IamInstanceProfile: !Ref Ec2IAMInstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y docker
          systemctl enable docker
          service docker start
          sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
          sudo chmod +x /usr/local/bin/docker-compose
          sudo usermod -a -G docker ec2-user
          sudo su - ec2-user


  Ec2RoleForNodes:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          -
            Effect: "Allow"
            Principal:
              Service:
                - "ec2.amazonaws.com"
            Action:
              - "sts:AssumeRole"
      Path: "/"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM
        - !Ref RkeClusterNodesPolicy

  RkeClusterNodesPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      Description: PolicyForRkeClusterNodes
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Resource: "*"
            Action:
              - "ec2:Describe*"
              - "ec2:AttachVolume"
              - "ec2:DetachVolume"
              - "ec2:*"
              - "elasticloadbalancing:*"


  Ec2IAMInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Path: "/"
      Roles:
        - !Ref Ec2RoleForNodes

```

ManagerサーバにSSM Session Managerで接続し、以下を実行します。
```sh
sudo su - ec2-user

# SSH鍵の配置
vim ~/.ssh/key.pem # CloudFormationで指定したキーペアの秘密鍵をコピー
chmod 400 ~/.ssh/key.pem

# RKEを取得
wget https://github.com/rancher/rke/releases/download/v0.1.17/rke
chmod +x rke

vim cluster.yml # cluster.ymlのaddressを各インスタンスのパブリックDNSに書き換えてコピー
./rke up
```

しばらく待つとクラスタの構築が完了し「Finished building Kubernetes custer successfully」と表示されます。  
`.kube_config_cluster.yml`が生成されるので、`~/.kube/config`にコピーします。
```sh
mkdir .kube
cp kube_config_cluster.yml ~/.kube/config
```

### kubectlのインストール
kuberctlはkubernetesクラスタを操作するためのCLIインターフェイス。  
設定ファイルは上記操作を行なった通り「~/.kube/config」に保存されます。  
下記を順に実行。

```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version
kubectl get nodes
kubectl config view
```

これでkuberctlによるkubernetes操作のセットアップは完了。
