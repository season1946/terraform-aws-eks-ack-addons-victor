
# Build Event-Driven Data Pipelines using AWS Controllers for Kubernetes (ACK) and Amazon EMR on EKS
In this example, we demonstrate to build an event-driven data pipeline using [AWS Controllers for Kubernetes (ACK)](https://aws-controllers-k8s.github.io/community/docs/community/overview/) and Amazon EMR on EKS. ACK is used to provision and configure serverless AWS resources: [Amazon EventBridge](https://aws.amazon.com/eventbridge/) and [AWS Step Functions](https://aws.amazon.com/step-functions/). Triggered by an Amazon EventBridge rule, AWS Step Functions orchestrates jobs running in Amazon EMR on EKS. By using ACK, you can use the Kubernetes API and configuration language to create and configure AWS resources the same way you create and configure a Kubernetes data processing jobs. The team can do the whole data operation without leaving the Kubernetes platform and only need to maintain the EKS cluster since all the other components are serverless.


The example demonstrates to build an event-driven data pipeline using ACK and Amazon EMR on EKS. Triggered by an Amazon EventBridge rule, AWS Step Functions orchestrates jobs running in Amazon EMR on EKS. 

[Code repo](https://github.com/season1946/terraform-aws-eks-ack-addons-victor/blob/demo/examples/event-driven-pipeline) for this example.

## Prerequisites:

Ensure that you have the following tools installed locally:

1. [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
2. [kubectl](https://Kubernetes.io/docs/tasks/tools/)
3. [terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)

## Deploy

To provision this example:

```bash
git clone -b demo https://github.com/season1946/terraform-aws-eks-ack-addons-victor.git
cd examples/event-driven-pipeline

region=<your region> # set region variable for following commands
terraform init
terraform apply -var region=$region #defaults to us-west-2
```

Enter `yes` at command prompt to apply

The following components are provisioned in your environment:
- A sample VPC, 3 Private Subnets and 3 Public Subnets
- Internet gateway for Public Subnets and NAT Gateway for Private Subnets
- EKS Cluster Control plane with one managed node group
- EKS Managed Add-ons: VPC_CNI, CoreDNS, Kube_Proxy, EBS_CSI_Driver
- K8S cluster autoscaler and fluentbit agent
- IAM execution roles for EMR on EKS, Step functions and EventBridge 

![terraform-output](img/terraform-output-eb-sfn-ack.png)

### Validate

The following command will update the `kubeconfig` on your local machine and allow you to interact with your EKS Cluster using `kubectl` to validate the deployment.

### Run `update-kubeconfig` command:

```bash
aws eks --region us-west-2 update-kubeconfig --name event-driven-pipeline-demo
```

### List the nodes

```bash
kubectl get nodes

# Output should look like below
NAME                                        STATUS   ROLES    AGE     VERSION
ip-10-1-10-64.us-west-2.compute.internal    Ready    <none>   19h     v1.24.9-eks-49d8fe8
ip-10-1-10-65.us-west-2.compute.internal    Ready    <none>   19h     v1.24.9-eks-49d8fe8
ip-10-1-10-7.us-west-2.compute.internal     Ready    <none>   19h     v1.24.9-eks-49d8fe8
ip-10-1-10-73.us-west-2.compute.internal    Ready    <none>   19h     v1.24.9-eks-49d8fe8
ip-10-1-11-96.us-west-2.compute.internal    Ready    <none>   19h     v1.24.9-eks-49d8fe8
ip-10-1-12-197.us-west-2.compute.internal   Ready    <none>   19h     v1.24.9-eks-49d8fe8
```

## Set up the event-driven pipeline
### Create an EMR virtual cluster
Let’s start by creating a [virtual cluster](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/virtual-cluster.html) in EMR and link it with a Kubernetes namespace in EKS. By doing that, the virtual cluster will use the linked namespace in EKS for hosting spark workloads.

Let's apply the manifest by using kubectl command below. Once done, you can go to *EMR console*, click *Virtual clusters* and you should see the cluster record as shown below.  
```bash
kubectl apply -f ack-yamls/emr-virtualcluster.yaml
```
![virtual cluster](img/vc-sfn.png)


### Create a S3 bucket and upload data
Next, let’s create a S3 bucket for storing spark pod templates and sample data. 
```bash
kubectl apply -f ack-yamls/s3.yaml
```
*Note*: If you don’t see the bucket got created, you can check the log from ACK S3 controller pod for details. The error is mostly caused by the bucket with the same name has existed. You need to change the bucket name in s3.yaml as well as in eventbridge.yaml and sfn.yaml. You also need to update upload-inputdata.sh and upload-spark-scripts.sh with the new bucket name.

Run the command below to upload input data and pod templates. Once done, sparkjob-demo-bucket S3 bucket is created with two folders: input and scripts.
```bash
bash spark-scripts-data/upload-inputdata.sh
```

### Create a Step Functions state machine

You need to make the following changes in sfn.yaml before apply. 

* replace the value for roleARN with stepfunctions_role_arn 
* replace the value for ExecutionRoleArn with emr_on_eks_role_arn
* replace the value for VirtualClusterId with your virtual cluster id
* optional: change sparkjob-demo-bucket with your bucket name 

sfn.yaml
```bash
apiVersion: sfn.services.k8s.aws/v1alpha1
kind: StateMachine
metadata:
  name: run-spark-job-ack
spec:
  name: run-spark-job-ack
  roleARN: "arn:aws:iam::xxxxxxxxxxx:role/event-driven-pipeline-demo-sfn-execution-role"   # replace with your stepfunctions_role_arn
  tags:
  - key: owner
    value: sfn-ack
  definition: |
      {
      "Comment": "A description of my state machine",
      "StartAt": "input-output-s3",
      "States": {
        "input-output-s3": {
          "Type": "Task",
          "Resource": "arn:aws:states:::emr-containers:startJobRun.sync",
          "Parameters": {
            "VirtualClusterId": "f0u3vt3y4q2r1ot11m7v809y6",  
            "ExecutionRoleArn": "arn:aws:iam::xxxxxxxxxxx:role/event-driven-pipeline-demo-emr-eks-data-team-a",
            "ReleaseLabel": "emr-6.7.0-latest",
            "JobDriver": {
              "SparkSubmitJobDriver": {
                "EntryPoint": "s3://sparkjob-demo-bucket/scripts/pyspark-taxi-trip.py",
                "EntryPointArguments": [
                  "s3://sparkjob-demo-bucket/input/",
                  "s3://sparkjob-demo-bucket/output/"
                ],
                "SparkSubmitParameters": "--conf spark.executor.instances=10"
              }
            },
            "ConfigurationOverrides": {
              "ApplicationConfiguration": [
                {
                 "Classification": "spark-defaults",
                "Properties": {
                  "spark.driver.cores":"1",
                  "spark.executor.cores":"1",
                  "spark.driver.memory": "10g",
                  "spark.executor.memory": "10g",
                  "spark.kubernetes.driver.podTemplateFile":"s3://sparkjob-demo-bucket/scripts/driver-pod-template.yaml",
                  "spark.kubernetes.executor.podTemplateFile":"s3://sparkjob-demo-bucket/scripts/executor-pod-template.yaml",
                  "spark.local.dir" : "/data1,/data2"
                }
              }
              ]
            }...
```
You can get your virtual cluster id from EMR console or use the command below.
```bash
kubectl get virtualcluster -o jsonpath={.items..status.id}
# result:
f0u3vt3y4q2r1ot11m7v809y6  # VirtualClusterId
```
Then, apply the manifest to create the step function state machine.
```bash
kubectl apply -f ack-yamls/sfn.yaml
```

### Create an EventBridge rule
The last step is to create an EventBridge rule. Let’s use the command below to the arn of the step function state machine created above.
```bash
kubectl get StateMachine -o jsonpath={.items..status.ackResourceMetadata.arn}
# result
arn: arn:aws:states:us-west-2:xxxxxxxxxx:stateMachine:run-spark-job-ack # sfn_arn
```
Then, update eventbridge.yaml with 

* replace the value for roleARN with eventbridge_role_arn
* replace with arn with your sfn_arn 
* optional: change sparkjob-demo-bucket with your bucket name 
eventbridge.yaml
```bash
apiVersion: eventbridge.services.k8s.aws/v1alpha1
kind: Rule
metadata:
  name: eb-rule-ack
  annotations:
    services.k8s.aws/region: us-west-2
spec:
  name: eb-rule-ack
  description: "ACK EventBridge Filter Rule to sfn using event bus reference"
  eventPattern: | 
    {
      "source": ["aws.s3"],
      "detail-type": ["Object Created"],
      "detail": {
        "bucket": {
          "name": ["sparkjob-demo-bucket"]    
        },
        "object": {
          "key": [{
            "prefix": "input/"
          }]
        }
      }
    }
  targets:
    - arn: arn:aws:states:us-west-2:xxxxxxxxxx:stateMachine:run-spark-job-ack # replace with your sfn arn
      id: sfn-run-spark-job-target
      roleARN: arn:aws:iam::xxxxxxxxx:role/event-driven-pipeline-demo-eb-execution-role # replace your eventbridge_role_arn
      retryPolicy:
        maximumRetryAttempts: 0 # no retries
  tags:
    - key: tag-01
      value: value-01
```

## Test the data pipeline
To test the data pipeline, we will trigger it by uploading a spark script to the S3 bucket scripts folder using the command below.
```bash
bash spark-scripts-data/upload-spark-scripts.sh
```
The upload event triggers the EventBridge event and then calls the step function state machine. As shown below, you can go to *Step Functions* console, click *State machines* and choose *run-spark-job-ack*. You will see a new execution is running. 

![sfn-results](img/sfn-results.png)


For the spark job details, you can go to *EMR console,* choose *Virtual clusters* and then click *my-ack-vc*. You will get all the job running history for this virtual cluster. Click *Spark UI* button in any row, you will be redirected the spark history server for more spark driver and executor logs.

![emr-results](img/emr-results.png)


## Destroy

To teardown and remove the resources created in this example:

```bash
aws s3 rm s3://sparkjob-demo-bucket --recursive # clean up data in S3

kubectl delete -f ack-yamls/. #Delete aws resources created by ACK

terraform destroy -target="module.eks_blueprints_kubernetes_addons" -target="module.eks_ack_addons" -auto-approve -var region=$region
terraform destroy -target="module.eks_blueprints" -auto-approve -var region=$region
terraform destroy -auto-approve -var region=$region
```
