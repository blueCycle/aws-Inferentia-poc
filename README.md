# Inferencing on Inf2 instances with vLLM running in EKS

The goal of this document is to deploy llama2-13b model on AWS Inferentia 2 instances on EKS. 

We are using the Neuron SDK 2.18.1 release that supports contineous batching. 

## Tables of Contents

1.  Prerequisites
2.  Create an Amazon EKS Cluster
3.  Lookup the Latest EKS Optimized Accelerated AMI
4.  Create an Inf2 EKS Managed Node Group
5.  Install the Neuron Device Plugin
6.  Install the Neuron Scheduling Extension
7.  Check the number of Neuron devices available in the cluster
8.  ECR Repository preparation
9.  Docker Image preparation and push to ECR
10. Llama2 13B Pod Deployment
11. Run the llmperf benchmark
12. Result for 4K context window
13. Result for 8k context window


## 1. Prerequisites

Ensure the following utilities are installed on your system. We will need these tools to interact with your AWS environemnt, EKS and build the docker image.

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
```
Deploy the cluster

```
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
```

Create the nodegroup

```
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
```
```
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
```
```
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
```
```
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
## 9. Docker Image preparation and push to ECR


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

WORKDIR /app/llmperf
```

The Docker container is expecting the directory `Inf2-eks-inference` in the same shared filesystem. The directory `inf2-eks-inference` should include `vllm` and `llmperf`.


Once we have the file we can build the docker container and upload it to the ECR repository

```
docker build -t neuron-container:pytorch .
```

Logging to the ECR repository created, tag the image and push the image to ECR:
```
aws ecr-public get-login-password --region us-east-1| docker login --username AWS --password-stdin public.ecr.aws/XXXXXX/neuron_2_18_1_repo
```
```
docker tag neuron-container:pytorch public.ecr.aws/XXXXXX/neuron_2_18_1_repo:latest
```
```
docker push public.ecr.aws/XXXXXX/neuron_2_18_1_repo:latest
```


## 10. Llama2 13B Pod Deployment

For this test we will request 8 Neuron cores per pod and set the server batch size to 24 for 4K context window. In this configuration we can run up to 3 pods simultaneously on an inf2.48xl instance. 

Build the pod deployment file.
```
cat > llama2-4k-tp8-b24.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: llama2-4k-tp8-b24
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
      command: ["python3", "-m", "vllm.entrypoints.openai.api_server", "--model=NousResearch/Llama-2-13b-chat-hf", "--tensor-parallel-size=8", "--max-num-seqs=24", "--max-model-len=4096", "--block-size=4096"]

      resources:
        limits:
          aws.amazon.com/neuron: 4
EOF
```
Deploy the pod

```
kubectl apply -f llama2-4k-tp8-b24.yaml
```
You can tail the logs by using the below command to check if its compiling well. 
```
kubectl logs -f llama2-4k-tp8-b24
```
Wait till you see the following output before you start with the llmperf benchmark test

```
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

## 11. Run the llmperf benchmark
```
kubectl exec -it llama2-4k-tp8-b8  -n default -- bash
```

cd /app/llmperf

Change the test parameters as needed on the latency-llama2-13b.sh file, see below steps for guidance.

Run the llmperf test.

```
bash latency-llama2-13b.sh
```
## 12. Results - llama2 13B with 4K context on Inf2 instances

This configuration uses 4 Neuron accelarators (tp=8, as each accelarator has 2 Neuron cores) and server batch on 24.

Below is the config used for llmperf test (latency-llama2-13b.sh file).

```
export LLM_PERF_CONCURRENT=22
export LLM_PERF_MAX_REQUESTS=$(expr ${LLM_PERF_CONCURRENT} \* 8 )
export OPENAI_API_KEY=EMPTY
export OPENAI_API_BASE="http://localhost:8000/v1“
export LLM_PERF_SCRIPT_DIR=/app/llmperf

export date_str=$(date '+%Y-%m-%d-%H-%M-%S')
export LLM_PERF_OUTPUT=outputs/${date_str}

mkdir -p $LLM_PERF_OUTPUT
cp "$0" "${LLM_PERF_OUTPUT}"/

python3 ${LLM_PERF_SCRIPT_DIR}/token_benchmark_ray.py \
--model "NousResearch/Llama-2-13b-chat-hf" \
--mean-input-tokens 3072 \
--stddev-input-tokens 200 \
--mean-output-tokens 512 \
--stddev-output-tokens 20 \
--max-num-completed-requests ${LLM_PERF_MAX_REQUESTS} \
--timeout 1800 \
--num-concurrent-requests ${LLM_PERF_CONCURRENT} \
--results-dir "${LLM_PERF_OUTPUT}" \
--llm-api openai \
--additional-sampling-params '{}'
```
****llmperf results**** 

**inter_token_latency_s = 0.109 msec.** This refers to the average time taken to generate each individual token in a sequence.

**ttft_s = 3.03 seconds.** Time to First Token - Is the time taken from the initiation of a request until the first token of output is generated.

**end_to_end_latency_s = 56.13 seconds.** This is the total time taken from the start of a request to the completion of the entire output. The equation for this is ttft + output tokens * inter token latency. 

**Overall Output Throughput = 166 tokens/second.** This indicates the rate at which tokens are produced on average across the entire test, measured in tokens per second.

Raw results below.

```
inter_token_latency_s
p25 = 0.10163148166861351
p50 = 0.10702191344503317
p75 = 0.11276160603150126
p90 = 0.11817865567186951
p95 = 0.11886552227389088
p99 = 0.1743002865265452
mean = 0.10989042557537512
min = 0.09626751026944404
max = 0.18549707034948917
stddev = 0.015529047546578849

ttft_s
p25 = 2.0387424847867806
p50 = 3.113982766517438
p75 = 4.074644652981078
p90 = 4.790316980972421
p95 = 4.983342077798443
p99 = 5.129886006060988
mean = 3.032657462526748
min = 0.5222592910286039
max = 5.1856156249996275
stddev = 1.3089841682680285

end_to_end_latency_s
p25 = 55.97674963020836
p50 = 57.4742788435542
p75 = 58.92187684998498
p90 = 60.478638478566424
p95 = 60.59555665924563
p99 = 64.4010606003867
mean = 56.13276988609207
min = 22.181874828995205
max = 65.64542450301815
stddev = 7.829863663852064

request_output_throughput_token_per_s
p25 = 8.885786169129272
p50 = 9.362075747530731
p75 = 9.858518232324709
p90 = 10.158432497068963
p95 = 10.306640215618996
p99 = 10.39981120369342
mean = 9.247944016363283
min = 5.461608204926722
max = 10.408046163675696
stddev = 0.9381490124267015

number_input_tokens
p25 = 2863.0
p50 = 3113.5
p75 = 3251.25
p90 = 3353.8
p95 = 3441.7000000000003
p99 = 3486.42
mean = 3076.590909090909
min = 2602
max = 3489
stddev = 234.8986241687329

number_output_tokens
p25 = 495.0
p50 = 511.5
p75 = 525.0
p90 = 535.1
p95 = 543.8
p99 = 556.83
mean = 494.5681818181818
min = 114
max = 565
stddev = 84.45238034214634

Number Of Errored Requests: 0
Overall Output Throughput: 166.55283788835882
Number Of Completed Requests: 44
Completed Requests Per Minute: 20.20584954851649
Launched Requests Per Minute: 35.987575600993196
```


