# Create and Run a Snowflake JDBC Source Connector for Confluent Cloud

This tutorial guides you through the process of setting up and running a Snowflake JDBC Source Connector for Confluent Cloud. This allows you to stream data from your Snowflake database into a Kafka topic on Confluent Cloud using the JDBC protocol.

## Prerequisites
Before starting this tutorial, you should have the following:

- Basic understanding of Snowflake, JDBC, Confluent Cloud, and Kubernetes.
- Access to a [Snowflake](https://www.snowflake.com/) warehouse/database.
- Access to a [Confluent Cloud](https://confluent.cloud/) Kafka cluster.
- A Kubernetes cluster set up and ready to use.
- [Confluent for Kubernetes](https://docs.confluent.io/operator/current/co-quickstart.html) installed on the Kubernetes cluster.
- Knowledge of how to work with YAML files.
- The following tools installed:
  - [kubectl](https://kubernetes.io/docs/reference/kubectl/)
  - A tool to calculate SHA512 checksums, such as [shasum](https://www.commandlinux.com/man-page/man1/shasum.1.html).

## Step 1: Download the JDBC Source Connector and the Snowflake JDBC Driver
First, you need to download two important files:
1. The Confluent JDBC Connector. You can download it from the [Confluent Hub](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc).  
2. The Snowflake JDBC driver. Download it from [Maven Repository](https://repo1.maven.org/maven2/net/snowflake/snowflake-jdbc/3.13.16/snowflake-jdbc-3.13.16.jar).

## Step 2: Add the Snowflake JDBC Driver to the JDBC Source Connector
This step involves modifying the Confluent JDBC Connector to include the Snowflake JDBC driver. Here's how you do it:
1. Extract the Confluent JDBC Connector zip file and navigate to the `lib` folder.
2. Copy the Snowflake JDBC driver JAR (`snowflake-jdbc-3.13.16.jar`) and paste it into this `lib` folder.
3. Compress the entire folder as a zip file - just as it was before you extracted it before. You now have a JDBC Connector plugin with the added Snowflake JDBC driver.
4. Note the SHA512 checksum of this newly created file (you will need this later).  
`shasum -a 512 '<your new zip file>'`

## Step 3: Upload the Updated JDBC Source Connector
1. Upload the new zip file to a location that can be accessed from your Kubernetes cluster where Kafka Connect is deployed. For example, you could use Amazon S3 or Google Cloud Storage.  
   - Alternatively, you may use the preconfigured zip file already uploaded [here](https://bucket-name.s3.us-east-2.amazonaws.com/confluentinc-kafka-connect-jdbc-snowflake-10.7.3.zip). Please note that this zip file contains a pre-configured version of the JDBC Source Connector with the Snowflake JDBC driver already added.


## Step 4: Create a Confluent Cloud API Key
1. To create an API key, log on to [Confluent Cloud](https://confluent.cloud/), navigate to your cluster, and create an API key. Make sure to download the key for later reference.  
![Create a Confluent Cloud API Key](/screenshots/API_Keys_Screenshot.png)


## Step 5: Create a Kubernetes secret with the Confluent Cloud API Key
In this step, we'll create a Kubernetes secret to store the API key.
1. Create a file named `plain.txt` and enter the API key and secret in the following format (replace `YOUR_API_KEY` and `YOUR_SECRET` with your actual API key and secret):
```
username=YOUR_API_KEY
password=YOUR_SECRET
```
2. Using `kubectl`, create a Kubernetes secret for the API key with the file you just created. This command will create a Kubernetes secret named `ccloud-credentials`, which will be used later for authentication.  
  `kubectl create secret generic ccloud-credentials --from-file=plain.txt`

## Step 6: Create and Apply YAML Manifest for Connect
1. Ensure that the Confluent for Kubernetes operator is installed on your Kubernetes cluster. For more information, refer to the [Confluent for Kubernetes Quickstart](https://docs.confluent.io/operator/current/co-quickstart.html).

2. Create a new YAML file for [Connect](https://docs.confluent.io/operator/current/co-api.html#tag/Connect), as shown below. Note the following:
    - The `build` section references the URL and SHA512 checksum for the modified JDBC connector you previously created. Replace the `archivePath` with the actual URL of your modified JDBC connector.
      - Alternatively, you may use the preconfigured zip file referenced in the `archivePath` and `checksum` values below. Please note that this zip file contains a pre-configured version of the JDBC Source Connector with the Snowflake JDBC driver already added. 
    - The `dependencies` section contains the bootstrap URL (as found in _Cluster Settings_ at https://confluent.cloud/), the secret containing the API key, and TLS settings for Connect to communicate with Confluent Cloud using the pre-installed TLS certificates in the image.
```
apiVersion: platform.confluent.io/v1beta1
kind: Connect
metadata:
  name: connect
  namespace: confluent
spec:
  replicas: 1
  image:
    application: confluentinc/cp-server-connect:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  build:
    type: onDemand
    onDemand:
      plugins:
        locationType: url
        url:
          - name: "kafka-connect-jdbc"
            archivePath: "https://gtrout-public.s3.us-east-2.amazonaws.com/confluentinc-kafka-connect-jdbc-snowflake-10.7.3.zip"
            checksum: "82cf0fe99a88318f56e37636da94af8ae24df0e89fc7e4c7c21bd459fb226a9bb95942db5a744e499ae1cd4bdf87e679a4976480a71f4e0c423ce9d88f28aa4f"
  dependencies:
    kafka:
      bootstrapEndpoint: "pkc-xxxxx.us-east-2.aws.confluent.cloud:9092"
      authentication:
        type: plain
        jaasConfig:
          secretRef: "ccloud-credentials"
      tls:
        enabled: true
        ignoreTrustStoreConfig: true 
```
3. Apply the manifest to create the Connect cluster by running the following command:  
   `kubectl apply -f connect.yaml`


## Step 7: Create and apply YAML File for the Snowflake JDBC Connector
At this point, you have a Connect cluster running and connected to your Confluent Cloud cluster. It has the Snowflake JDBC Source Connector plugin installed, which means a connector of this type can be deployed. The next step is to configure the Snowflake JDBC Source connector, referencing all the relevant Snowflake configurations such as `connect.url`, `warehouse`, and others. If needed, you can refer to Snowflake's [JDBC Driver Connection Parameter Reference](https://docs.snowflake.com/developer-guide/jdbc/jdbc-parameters) page for more details.

1. Create a new YAML file for the [Connector](https://docs.confluent.io/operator/current/co-api.html#tag/Connector), as shown below:
```
apiVersion: platform.confluent.io/v1beta1
kind: Connector
metadata:
  name: snowflake-jdbc-source
  namespace: confluent
spec:
  class: "io.confluent.connect.jdbc.JdbcSourceConnector"
  taskMax: 1
  connectClusterRef:
    name: connect
  configs:
    connection.url: "jdbc:snowflake://xxxxxxx-xxxxxxx.snowflakecomputing.com/?warehouse=EXAMPLEWAREHOUSE&db=EXAMPLEDB&role=PUBLIC&schema=PUBLIC&user=USER123&password=password&tracing=ALL"
    table.whitelist: "FOO"
    mode: "timestamp+incrementing"
    timestamp.column.name: "UPDATE_TS"
    incrementing.column.name: "ID"
    topic.prefix: "snowflake-"
    validate.non.null: "false"
    errors.log.enable: "true"
    errors.log.include.messages: "true"
```
2. Apply the manifest to create the connector:  
   `kubectl apply -f connector.yaml`

Remember to replace the placeholders in the `connection.url` field with your actual Snowflake details. The `table.whitelist` field should contain the name of the table you want to use. The `timestamp.column.name` and `incrementing.column.name` fields should contain the names of the timestamp and incrementing columns, respectively. The `topic.prefix` field will be used as a prefix for the names of the topics where the data will be published. The `errors.log.enable` and `errors.log.include.messages` fields are used to enable error logging and to include error messages in the logs, respectively.

## _Optional_ - Step 8: Configure Confluent Control Center 
Confluent Control Center provides a self-hosted Web UI for monitoring and/or managing your Connect cluster and other Confluent components, including Confluent Cloud Kafka clusters.  
![Viewing Connectors in Confluent Control Center](/screenshots/Control_Center_Screenshot.png)

If you wish to set it up:
1. Create a new YAML file for [Control Center](https://docs.confluent.io/operator/current/co-api.html#tag/ControlCenter), as shown below.
    - The `kafka` dependency can be removed if you only need Control Center to manage the Connect cluster. In this example, the dependency is included and it uses the same API key/secret that was used by the Connect cluster.
    - The `externalAccess` section is only needed if you want to use a load balancer to access Control Center externally. If you want to get started quickly or if your Kubernetes cluster is not configured to provision `LoadBalancer` services, you can remove this entire section. Instead, you can access Control Center at `localhost:9021` in your local browser by running the following command:
      `kubectl port-forward controlcenter-0 9021:9021` 
      - Replace `your-domain.net` with the domain where your Kubernetes cluster is running. You might be able to enter any value here and still access Control Center using the external IP address assigned to the Control Center service. To retrieve this IP address, you can execute the following command:  
    `kubectl get service controlcenter-bootstrap-lb`
```
apiVersion: platform.confluent.io/v1beta1
kind: ControlCenter
metadata:
  name: controlcenter
  namespace: confluent
spec:
  replicas: 1
  image:
    application: confluentinc/cp-enterprise-control-center:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  dataVolumeCapacity: 10Gi
  configOverrides:
    server:
    - confluent.metrics.topic.max.message.bytes=8388608  
  dependencies:
    kafka:
      bootstrapEndpoint: "pkc-xxxxx.us-east-2.aws.confluent.cloud:9092"
      authentication:
        type: plain
        jaasConfig:
          secretRef: "ccloud-credentials"
      tls:
        enabled: true
        ignoreTrustStoreConfig: true
    connect:
    - name: connect
      url: http://connect.confluent.svc.cluster.local:8083
  externalAccess:
    type: loadBalancer
    loadBalancer:
      domain: your-domain.net
```

## Troubleshooting
When troubleshooting, ensure that your kubectl context is set to the `confluent` namespace (or the namespace where your resources are deployed). To set the namespace for all subsequent kubectl commands in your context permanently, run the following command:  
`kubectl config set-context --current --namespace=confluent`


### Checking the Status of Resources
To check the status of all Confluent [custom resources](https://docs.confluent.io/operator/current/co-configure-overview.html#kubernetes-custom-resources-for-cp), use the following command:  
`kubectl get confluent`

### Describing Resources
To get detailed descriptions of the Connect, Connector, or ControlCenter custom resources, use the following commands:

`kubectl describe connect connect`  
`kubectl describe connector snowflake-jdbc-source`  
`kubectl describe controlcenter controlcenter`  
_Note: The names of the Connect, Connector, and ControlCenter resources are connect, snowflake-jdbc-source, and controlcenter, respectively._

### Viewing Logs
To view the logs of the Confluent for Kubernetes pod, Connect pod, ControlCenter pod, and Connect init container's pod, use the following commands:

Confluent for Kubernetes pod logs:  
`kubectl logs confluent-operator-xxxxxxxxxx-xxxxx`

Connect pod logs:  
`kubectl logs connect-0`

ControlCenter pod logs:  
`kubectl logs controlcenter-0`

Connect init container's pod logs (useful if the connector plugin is not available):  
`kubectl logs connect-0 -c config-init-container`

### Viewing Events
To view all events in the namespace, sorted by timestamp, use the following command:  
`kubectl get events --sort-by=.metadata.creationTimestamp`
