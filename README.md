# Inferencing on Inf2 instances with vLLM running in EKS
The goal of this document is to deploy llama2-13b model in a multitenancy setup with EKS and Inf2 instances.

## Tables of Contents

1.  Prerequisites
2.  Create an Amazon EKS Cluster
3.  Lookup the Latest EKS Optimized Accelerated AMI
4.  Create an Inf2 EKS Managed Node Group
5.  Install the Neuron Device Plugin
6.  Install the Neuron Scheduling Extension
7.  Check the number of Neuron devices available in the cluster
8.  ECR Repository preparation
9.  Docker Image preparation 
10. Llama2 13B Pod Deployment through vllm
11. Run the llmperf benchmark


## 1. Prerequisites

Ensure the following utilities are installed:

1. [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
2. [eksctl](https://eksctl.io/introduction/#installation)
3. [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
4. [Helm v3](https://helm.sh/docs/intro/install/)

Start by setting some variables for your environment. Replace the values to suit your environment.

```
export AWS_REGION=us-east-2
export CLUSTER_NAME=llama2
export EKS_VERSION=1.29
```

## 2. Create an Amazon EKS Cluster

If you already have an EKS cluster, you can skip this step.

You can use `eksctl` to create an Amazon EKS cluster. Feel free to change the configuration below to meet your needs. You will most likely want to change the instance type and desired capacity.

```
cat > eks_cluster.yaml <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: $CLUSTER_NAME
  region: $AWS_REGION
  version: "$EKS_VERSION"

addons:
- name: vpc-cni
  version: latest

cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
    
iam:
  withOIDC: true
EOF

eksctl create cluster --config-file eks_cluster.yaml
```

It will take approximately 15-20 minutes to create the EKS control plane and “main” node group.

References:

* `eksctl` Docs: [Creating and managing clusters](https://eksctl.io/usage/creating-and-managing-clusters/)
* `eksctl` Docs: [Addons](https://eksctl.io/usage/addons/)

## 3. Lookup the Latest EKS Optimized Accelerated AMI

The Amazon EKS optimized accelerated Amazon Linux AMI is built on top of the standard Amazon EKS optimized Amazon Linux AMI. It's configured to serve as an optional image for Amazon EKS nodes to support GPU and [Inferentia](http://aws.amazon.com/machine-learning/inferentia/) based workloads.

You can programmatically retrieve the Amazon Machine Image (AMI) ID for Amazon EKS optimized AMIs by querying the AWS Systems Manager Parameter Store API using the command below.

```
export ACCELERATED_AMI=$(aws ssm get-parameter \
    --name /aws/service/eks/optimized-ami/$EKS_VERSION/amazon-linux-2-gpu/recommended/image_id \
    --region $AWS_REGION \
    --query "Parameter.Value" \
    --output text)
```

References:

* EKS Docs: [Retrieving Amazon EKS optimized Amazon Linux AMI IDs](https://docs.aws.amazon.com/eks/latest/userguide/retrieve-ami-id.html)
* EKS Docs: [Amazon EKS optimized Amazon Linux AMIs](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)
* GitHub: [awslabs / amazon-eks-ami](https://github.com/awslabs/amazon-eks-ami)

## 4. Create an Inf2 EKS Managed Node Group

Start by setting some variables for your node group. Choose the right combination of instance size and number of nodes to meet your individual use case.

```
export INSTANCE_TYPE=inf2.48xlarge
export DESIRED_NODES=1
```

Then use `eksctl` to create the node group.

```
cat > eks_nodegroup.yaml <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: $CLUSTER_NAME
  region: $AWS_REGION
  version: "$EKS_VERSION"
    
managedNodeGroups:
- name: neuron
  ami: $ACCELERATED_AMI
  amiFamily: AmazonLinux2
  iam:
    attachPolicyARNs:
    - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
    - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
    - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
    - arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
  instanceType: $INSTANCE_TYPE
  overrideBootstrapCommand: |
     #!/bin/bash
     /etc/eks/bootstrap.sh $CLUSTER_NAME
  desiredCapacity: $DESIRED_NODES
  volumeSize: 1024
  ssh:
     allow: true
     publicKeyPath: ~/.ssh/id_rsa.pub
EOF

eksctl create nodegroup \
    --config-file eks_nodegroup.yaml \
    --install-neuron-plugin=false
```

References:

* `eksctl` Docs: [Managing nodegroups](https://eksctl.io/usage/managing-nodegroups/)
* `eksctl` Docs: [EKS Managed Nodegroups](https://eksctl.io/usage/eks-managed-nodes/)
* `eksctl` Docs: [Custom AMI support](https://eksctl.io/usage/custom-ami-support/)

## 5. Install the Neuron Device Plugin

The Neuron Device Plugin exposes Neuron cores & devices to Kubernetes as the following resources

* [`aws.amazon.com/neuroncor`](http://aws.amazon.com/neuroncore)`e` — used for allocating neuron cores to the container
* [`aws.amazon.com/neurondevice`](http://aws.amazon.com/neurondevice) —  used for allocating neuron devices to the container and is the recommended resource for allocating devices to the container
* `[aws.amazon.com/neuron](http://aws.amazon.com/neuron)` — also allocates neuron devices and this exists just to be backward compatible with already existing installations

Use `kubectl` to install the Neuron Device Plugin. 

```
kubectl apply -f https://raw.githubusercontent.com/aws-neuron/aws-neuron-sdk/master/src/k8/k8s-neuron-device-plugin-rbac.yml
kubectl apply -f https://raw.githubusercontent.com/aws-neuron/aws-neuron-sdk/master/src/k8/k8s-neuron-device-plugin.yml
```

References:

* Neuron Docs: [Containers - Kubernetes - Getting Started](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/containers/kubernetes-getting-started.html#k8s-neuron-device-plugin)
* Neuron Docs: [Kubernetes environment setup for Neuron](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/containers/tutorials/k8s-setup.html#k8s-neuron-device-plugin)

## 6. Install the Neuron Scheduling Extension

The Neuron scheduler extension is required for scheduling pods that require more than one Neuron core or device resource. The Neuron scheduler extension filter out nodes with non-contiguous core/device ids and enforces allocation of contiguous core/device ids for the Pods requiring it.

Use `kubectl` to install the Neuron Scheduling Extension. 

```
kubectl apply -f https://raw.githubusercontent.com/aws-neuron/aws-neuron-sdk/master/src/k8/k8s-neuron-scheduler-eks.yml
kubectl apply -f https://raw.githubusercontent.com/aws-neuron/aws-neuron-sdk/master/src/k8/my-scheduler.yml
```


References:

* Neuron Docs: [Containers - Kubernetes - Getting Started](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/containers/kubernetes-getting-started.html#k8s-multiple-scheduler)
* Neuron Docs: [Kubernetes environment setup for Neuron](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/containers/tutorials/k8s-setup.html#k8s-multiple-scheduler)

## 7. Check the number of Neuron devices available in the cluster

Before we can run our pods in the cluster we need to be sure the Neuron SDK plugging has been installed and we can see the neuron cores available

Verify that neuron device plugin is running: 
```
kubectl get ds neuron-device-plugin-daemonset --namespace kube-system
```

Verify that the node has allocatable neuron cores and devices
```
kubectl get nodes "-o=custom-columns=NAME:.metadata.name,NeuronCore:.status.allocatable.aws\.amazon\.com/neuroncore"
```

## 8. ECR Repository preparation 

Create a ECR repository

```
export ECR_REPO_NAME='neuron_2_18_1_repo'
aws ecr-public create-repository --repository-name $ECR_REPO_NAME --region us-east-1
```
If the repository already exists, check it using this command:
```
aws ecr-public describe-repositories --repository-name $ECR_REPO_NAME --region us-east-1
```

The output is as below:

```
{                                                                                                                                                                                                
    "repository": {                                                                                                                                                                              
        "repositoryArn": "arn:aws:ecr-public::<account ID>:repository/neuron_2_18_1_repo",                                                                                                      
        "registryId": "<account ID>",                                                                                                                                                            
        "repositoryName": "neuron_2_18_1_repo",                                                                                                                                                 
        "repositoryUri": "public.ecr.aws/<XXXXXXX>/neuron_2_18_1_repo",                                                                                                                          
        "createdAt": 1698108079.114                                                                                                                                                              
    },                                                                                                                                                                                           
    "catalogData": {}                                                                                                                                                                            
}                                                  

```
## 9. Docker Image preparation 


For the Docker Image preparation we will use a AWS Deep Learning Container already prebuild with the latest Neuron SDK.  To be able to use the AWS DL image we need to login first
```
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 763104351884.dkr.ecr.us-west-2.amazonaws.com
```

Now we can prepare a Dockerfile with the AWS Deep Learining Container. 

Download and unzip the tarball of `inf2-eks-inference.tar.gz` in the same working folder.

Create a Dockerfile


```
FROM 763104351884.dkr.ecr.us-west-2.amazonaws.com/pytorch-inference-neuronx:2.1.2-neuronx-py310-sdk2.18.1-ubuntu20.04
WORKDIR /app
COPY ./Inf2-eks-inference /app
WORKDIR /app/vllm

# Add your custom stack of code
#RUN git clone https://github.com/aws-neuron/aws-neuron-samples.git
RUN pip install sentencepiece transformers==4.36.2 -U
RUN pip install transformers-neuronx --extra-index-url=https://pip.repos.neuron.amazonaws.com -U
RUN python -m pip install outlines
RUN pip install -r requirements-neuron.txt
RUN pip install sentencepiece transformers==4.36.2 -U
RUN pip install mpmath==1.3.0
RUN pip install -U numba
RUN pip install -e .

WORKDIR /app/llmperf
RUN pip install -e .
RUN pip install protobuf==3.19.2
RUN pip install fschat

WORKDIR /app/vllm
```

The Docker container is expecting the directory `Inf2-eks-inference` in the same shared filesystem. The directory `inf2-eks-inference` should include `vllm_llmperf` and `llmperf`.


Once we have the file we can build the docker container and upload it to the ECR repository

```
docker build -t neuron-container:pytorch .
```

Logging to the ECR repository created, tag the image and push the image to ECR:
```
aws ecr-public get-login-password --region us-east-1| docker login --username AWS --password-stdin public.ecr.aws/XXXXXX/neuron_2_18_1_repo
docker tag neuron-container:pytorch public.ecr.aws/XXXXXX/neuron_2_18_1_repo:latest
docker push public.ecr.aws/XXXXXX/neuron_2_18_1_repo:latest
```


## 10. Llama2 13B Pod Deployment through vllm

For this test we will request 8 Neuron cores per pod, so we can run up to 3 pods simultaneously. 

The POD is expecting to have Llama2-13b directory and 

```
cat > llama2-4k-tp8-b8.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: llama2-4k-tp8-b8
  labels:
    app.kubernetes.io/name: proxy
spec:
  restartPolicy: Never
  schedulerName: my-scheduler
  nodeSelector:
    node.kubernetes.io/instance-type: inf2.48xlarge
  containers:
    - name: llama2-inf
      image: "public.ecr.aws/xxxxxx/neuron_2_18_1_repo"
      imagePullPolicy: Always
      ports:
        - containerPort: 8080
          name: http-web-svc
      command: ["python3", "-m", "vllm.entrypoints.openai.api_server", "--model=NousResearch/Llama-2-13b-chat-hf", "--tensor-parallel-size=8", "--max-num-seqs=8", "--max-model-len=4096", "--block-size=4096"]

      resources:
        limits:
          aws.amazon.com/neuron: 4
EOF

kubectl apply -f llama2-4k-tp8-b8.yaml
```

## 11. Run the llmperf benchmark
```
kubectl exec -it llama2-4k-tp8-b8  -n default -- bash
```

cd /app/llmperf

Change the test parameters as needed on the latency-llama2-13b.sh file.

Run the llmperf test.

```
bash latency-llama2-13b.sh
```



