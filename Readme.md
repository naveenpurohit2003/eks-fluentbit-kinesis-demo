Create an Amazon EKS cluster named fluent-bit-demo using eksctl, as shown in the EKS docs, and then create a policy file called eks-fluent-bit-daemonset-policy.json (source) with the following content:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "firehose:PutRecordBatch"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "logs:PutLogEvents",
            "Resource": "arn:aws:logs:*:*:log-group:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:DescribeLogStreams",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:log-group:*"
        },
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "*"
        }
    ]
}
JSON
To attach this policy file to the EKS on EC2 worker nodes, execute the following sequence:

$ STACK_NAME=$(eksctl get nodegroup --cluster fluent-bit-demo -o json | jq -r '.[].StackName')

$ INSTANCE_PROFILE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceProfileARN") | .OutputValue')

$ ROLE_NAME=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceRoleARN") | .OutputValue' | cut -f2 -d/)

$ aws iam put-role-policy \
    --role-name $ROLE_NAME \
    --policy-name FluentBit-DS \
    --policy-document file://eks-fluent-bit-daemonset-policy.json
Bash
And now we move on to defining the Kubernetes RBAC settings – that is, the service account the Fluent Bit pods will be using along with the role and role binding.

First create the service account fluent-bit (this is what we will later use in the daemon set) by executing kubectl create sa fluent-bit.

Next, define the role and binding in a file named eks-fluent-bit-daemonset-rbac.yaml(source):

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: pod-log-reader
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: pod-log-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-log-reader
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: default
YAML
Now, in order to make the access permissions for the Fluent Bit plugin effective, you create the role and role binding, defined above, by executing the command kubectl apply -f eks-fluent-bit-daemonset-rbac.yaml.

In contrast to the ECS case, where we backed the configuration into a custom image, in our Kubernetes setup we’re using a config map to define the log parsing and routing for the Fluent Bit plugin. For this, use a file called eks-fluent-bit-configmap.yaml(source) with the following content:

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  labels:
    app.kubernetes.io/name: fluentbit
data:
  fluent-bit.conf: |
    [SERVICE]
        Parsers_File  parsers.conf
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10
    [FILTER]
        Name parser
        Match **
        Parser nginx
        Key_Name log
    [OUTPUT]
        Name firehose
        Match **
        delivery_stream eks-stream
        region us-west-2 
  parsers.conf: |
    [PARSER]
        Name   nginx
        Format regex
        Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")? \"-\"$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z
YAML
Create this config map by executing the command kubectl apply -f eks-fluent-bit-configmap.yaml and then define the Kubernetes Daemonset (using said config map) in a file called eks-fluent-bit-daemonset.yaml (source) with below content:

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentbit
  labels:
    app.kubernetes.io/name: fluentbit
spec:
  selector:
    matchLabels:
      name: fluentbit
  template:
    metadata:
      labels:
        name: fluentbit
    spec:
      serviceAccountName: fluent-bit
      containers:
      - name: aws-for-fluent-bit
        image: amazon/aws-for-fluent-bit:1.2.0
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
        - name: mnt
          mountPath: /mnt
          readOnly: true
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 500m
            memory: 100Mi
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      - name: mnt
        hostPath:
          path: /mnt
YAML
Finally, launch the Fluent Bit daemonset by executing kubectl apply -f eks-fluent-bit-daemonset.yaml and verify the Fluent Bit daemonset by peeking into the logs like so:

$ kubectl logs ds/fluentbit
Found 3 pods, using pod/fluentbit-9zszm
Fluent Bit v1.1.3
Copyright (C) Treasure Data

[2019/07/08 13:44:54] [ info] [storage] initializing...
[2019/07/08 13:44:54] [ info] [storage] in-memory
[2019/07/08 13:44:54] [ info] [storage] normal synchronization mode, checksum disabled
[2019/07/08 13:44:54] [ info] [engine] started (pid=1)
[2019/07/08 13:44:54] [ info] [in_fw] listening on unix:///var/run/fluent.sock
...
[2019/07/08 13:44:55] [ info] [sp] stream processor started
Bash
Next, deploy the following NGINX app via kubectl apply -f eks-nginx-app.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: nginx
spec:
  replicas: 4 
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
YAML
With that, we’re done setting up the log sources and routing. Now let’s move on to actually doing something with all the log data we’re collecting from the NGINX containers running in ECS and EKS: we will perform a centralized analysis of the logs.

Log analysis across clusters
The goal is to do a log analysis of the NGINX containers running in the ECS and EKS clusters. For this, we’re using Amazon Athena, which allows us to interactively query the service log data from Amazon S3 using SQL. Before we can query the data in S3, however, we need to get the log data there.

Remember that in the Fluent Bit configurations for ECS and EKS (above) we set the output to delivery_stream xxx-stream. That’s an Amazon Kinesis Firehose delivery stream, and we first have to create it, for ECS and EKS.

First, set up the access control part by defining a policy that effective allows Firehose to write to S3. To do this, we need to create a new IAM Role with two policy files. First, firehose-policy.json(source):

{
  "Version": "2012-10-17",
  "Statement": {
      "Effect": "Allow",
      "Principal": {
        "Service": "firehose.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
  }
}
JSON
Second, in the firehose-delivery-policy.json policy file (source), replace the XXXXXXXXXXXX with your own account ID (if you’re unsure what it is, you can get the account ID  by executing aws sts get-caller-identity --output text --query 'Account'). Also, in the S3 section, replace mh9-firelens-demo with your own bucket name.

Now we can create the firehose_delivery_role to use for both the ECS and the EKS delivery streams:

$ aws iam create-role \
        --role-name firehose_delivery_role \
        --assume-role-policy-document file://firehose-policy.json
Bash
From the resulting JSON output of the above command, note down the role ARN, which will be something in the form of arn:aws:iam::XXXXXXXXXXXXX:role/firehose_delivery_role. We will use this soon to create the delivery stream, but before that can happen we have to put in place the policy defined in the firehose-delivery-policy.json:

$ aws iam put-role-policy \
        --role-name firehose_delivery_role \
        --policy-name firehose-fluentbit-s3-streaming \
        --policy-document file://firehose-delivery-policy.json
Bash
Now create the ECS delivery stream:

$ aws firehose create-delivery-stream \
            --delivery-stream-name ecs-stream \
            --delivery-stream-type DirectPut \
            --s3-destination-configuration \
RoleARN=arn:aws:iam::XXXXXXXXXXXX:role/example_firehose_delivery_role,\
BucketARN="arn:aws:s3:::mh9-firelens-demo",\
Prefix=ecs
Bash
NOTE The spacing in above command matters: RoleARN etc. must be on one line without spaces.

Now we have to repeat the above for the EKS delivery stream, re-using the role created in the first step. (In other words, you only need to repeat the aws firehose create-delivery-stream command replacing ecs-stream with eks-streamand Prefix=ecs with Prefix=eks.)

It will take a couple of minutes for the delivery streams to be created and active. When you see something like the following, you’re ready to move on to the next step:

Amazon Kinesis Firehose delivery streams

We now need to generate some load for the NGINX containers running in ECS and EKS. You can grab the load generator files for ECS  and EKS and execute the commands below; this will curl the respective NGINX services every two seconds (executing in the background), until you kill the scripts:

$ ./load-gen-ecs.sh &
$ ./load-gen-eks.sh &
Bash
Now that we have some log data from the NGINX webservers, we can query the log entries in S3 from Athena. For this, we first have to create tables for ECS and EKS, telling Athena about the schema we’re using (here shown for the ECS log data and the same applies for EKS):