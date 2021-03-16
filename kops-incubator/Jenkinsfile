def USER_INPUT = input(
                        message: 'Kops AWS Setup, Choose AWS Region and HA for Kuberentes Cluster',
                        parameters: [
                            [$class: 'ChoiceParameterDefinition',
                            choices: ['us-east-1','us-east-2','us-west-1','us-west-2','af-south-1','ap-east-1', 'ap-south-1','ap-northeast-1', 'ap-northeast-3', 'ap-norhteast-2', 'ap-southeast-1', 'ap-southeast-2'].join('\n'),
                            name: 'awsregion',
                            description: 'Choose AWS Region for S3 and Kubernetes Cluster'],
                            [$class: 'ChoiceParameterDefinition',
                            choices: ['yes','no'].join('\n'),
                            name: 'enableHA',
                            description: 'Enable HA on Kubernetes Cluster']
                            ])
pipeline {
    agent any

    stages {
        stage('awsinstall') {
            steps {
                script {
                    sh '''#!/bin/bash
                    curl 'https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip' -o 'awscliv2.zip'
                    unzip -qq awscliv2.zip
                    sudo rm -rf /usr/local/bin/aws /usr/local/bin/aws_completer /usr/local/aws-cli
                    sudo ./aws/install
                    rm -rf ./aws* && aws iam list-users
                    '''
                }
            }
        }
        stage('kOpsinstall') {
            steps {
                script {
                    sh '''#!/bin/bash
                    #curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
                    #chmod +x ./kops
                    #sudo mv ./kops /usr/local/bin/
                    curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
                    chmod +x ./kubectl
                    sudo mv ./kubectl /usr/local/bin/kubectl
                    '''
                }
            }
        }
        stage('AWS IAM Configure') {
            steps {
                script {
                    sh '''#!/bin/bash
                    iam=$(aws iam get-group --group-name kops | jq -r '.Group.GroupName' | tr -d '\n')
                    if [[ $iam == "kops" ]]; then
                        echo "IAM Roles already exists for Kops"
                    else

                    aws iam create-group --group-name kops

                    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
                    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
                    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
                    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
                    aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops

                    aws iam create-user --user-name kops

                    aws iam add-user-to-group --user-name kops --group-name kops

                    aws iam create-access-key --user-name kops
                    fi
                    '''
                }
            }
        }
        stage('AWS Kops S3 Storage') {
            steps {
                script {
                    if ("${USER_INPUT['awsregion']}" == "us-east-1"){
                        sh """#!/bin/bash
                        echo \"${USER_INPUT}\"
                        bu=\$(aws s3api list-buckets | jq -r '.Buckets[].Name' | grep "prefix-kops-k8s-local-state-store" | tr -d '\n')
                        if [[ \$bu == "prefix-kops-k8s-local-state-store" ]]; then
                            echo "Kops S3 bucket already exists"
                        else
                            aws s3api create-bucket --bucket prefix-kops-k8s-local-state-store --region \"${USER_INPUT['awsregion']}\"
                            #Cluster State storage
                            aws s3api put-bucket-versioning --bucket prefix-kops-k8s-local-state-store  --versioning-configuration Status=Enabled
                            #Using S3 default bucket encryption
                            aws s3api put-bucket-encryption --bucket prefix-kops-k8s-local-state-store --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
                        fi
                        """
                    } else {
                        sh """#!/bin/bash
                        echo \"${USER_INPUT}\"
                        bu=\$(aws s3api list-buckets | jq -r '.Buckets[].Name' | grep "prefix-kops-k8s-local-state-store" | tr -d '\n')
                        if [[ \$bu == "prefix-kops-k8s-local-state-store" ]]; then
                            echo "Kops S3 bucket already exists"
                        else
                            aws s3api create-bucket --bucket prefix-kops-k8s-local-state-store --region \"${USER_INPUT['awsregion']}\" --create-bucket-configuration LocationConstraint=\"${USER_INPUT['awsregion']}\"
                            #Cluster State storage
                            aws s3api put-bucket-versioning --bucket prefix-kops-k8s-local-state-store  --versioning-configuration Status=Enabled
                            #Using S3 default bucket encryption
                            aws s3api put-bucket-encryption --bucket prefix-kops-k8s-local-state-store --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
                        fi
                        """
                    }
                }
            }
        }
        stage('Generate SSH Keys') {
            steps {
                sh '''#!/bin/bash
                echo 'y' | ssh-keygen -b 2048 -t rsa -f ~/.ssh/id_rsa -q -N ""
                '''
            }
        }
        stage('Kops Cluster Setup') {
            steps {
                script {
                    if( "${USER_INPUT['enableHA']}" == "yes"){
                        sh '''#!/bin/bash
                        export NAME="kops.k8s.local"
                        zones=\$(aws ec2 describe-availability-zones --region \"${USER_INPUT['awsregion']}\" | jq -r '.AvailabilityZones[].ZoneName' | wc -l)
                        if [[ $zones == 3 ]] || [[ $zones -gt 3 ]]; then
                            zonelist=\$(aws ec2 describe-availability-zones --region \"${USER_INPUT['awsregion']}\" | jq -r '.AvailabilityZones[].ZoneName' | head -n 3 | tr '\n' ',' | sed 's/.$//')
                            echo \$zonelist
                            echo Kops Cluster name \$NAME
                            kops create cluster --node-count 3 --zones $zonelist --master-zones $zonelist --node-size t3a.xlarge --master-size t2.medium \$NAME
                            kops update cluster \$NAME --yes
                        else
                            echo \"${USER_INPUT['awsregion']} doesn't have 3 AZ's\"
                        fi
                        '''
                    } else {
                        sh """#!/bin/bash
                        echo "one cluster"
                        export KOPS_STATE_STORE="s3://prefix-kops-k8s-local-state-store"
                        export NAME="kops.k8s.local"
                        echo \"${USER_INPUT['awsregion']}\"
                        aws ec2 describe-availability-zones --region \"${USER_INPUT['awsregion']}\" | jq -r '.AvailabilityZones[].ZoneName' | head -n 1 | tr -d '\n'
                        zonelist=\$(aws ec2 describe-availability-zones --region \"${USER_INPUT['awsregion']}\" | jq -r '.AvailabilityZones[].ZoneName' | head -n 1 | tr -d '\n')
                        echo \$zonelist
                        kops create cluster --zones \$zonelist \$NAME
                        kops create secret --name kops.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub
                        kops update cluster \$NAME --yes
                        """
                    }
                }
            }
        }
        stage('Validation') {
            steps {
                script {
                    sh '''#!/bin/bash
                    echo "Waiting While Kubernetes is bootstrapping"
                    sleep 600
                    export KOPS_STATE_STORE="s3://prefix-kops-k8s-local-state-store"
                    kops export kubecfg --admin
                    kubectl get nodes 
                    kops validate cluster --wait 10m
                    '''
                }
            }
        }
    }
}
