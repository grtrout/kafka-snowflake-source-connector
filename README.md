# Create and run a Snowflake JDBC Source Connector for Confluent Cloud

## Download the the JDBC Source Connector and the Snowflake JDBC driver
- Download the Confluent JDBC Connector from [Confluent Hub](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc).  

- Download the Snowflake JDBC driver.  
https://repo1.maven.org/maven2/net/snowflake/snowflake-jdbc/3.13.16/snowflake-jdbc-3.13.16.jar

## Add the Snowflake JDBC driver to the JDBC Source Connector
- Extract the Confluent JDBC Connector zip file and navigate to the lib folder.

- Copy the Snowflake JDBC driver JAR (`snowflake-jdbc-3.13.16.jar`) and paste it into this lib folder.

- Compress the entire folder as a zip file - just as it was before you extracted it before. You now have a JDBC Connector plugin with the added Snowflake JDBC driver.

- Note the SHA512 checksum of this newly created file (you will need this later).  
`shasum -a 512 '<your new zip file>'`

## Upload the updated JDBC Source Connector
- Upload this new zip file to a location that can be reached from the Kubernetes cluster where Kafka Connect is deployed - e.g., Amazon S3, Google Cloud Storage, etc.
  - You may also use the preconfigured zip fle uploaded here (assuming the file is still hosted at the time you are reading this):  
    https://bucket-name.s3.us-east-2.amazonaws.com/confluentinc-kafka-connect-jdbc-snowflake-10.7.3.zip


## Create a Confluent Cloud API Key
- Log on to [Confluent Cloud](https://confluent.cloud/), navigate to your cluster, and create an API key. Optionally, download the key for later reference.

## Create a Kubernetes secret with the Confluent Cloud API Key
- Create a file named `plain.txt` and enter the API key and secret in the following format:
```
username=<API Key>
password=<API Secret>
```

- Using _kubectl_, create a Kubernetes secret for the API key with the file you just created. The Kubernetes secret, named `ccloud-credentials`, will be used later to authenticate to Confluent Cloud.  
`kubectl create secret generic ccloud-credentials --from-file=plain.txt`

## Create YAML manifest for Kafka Connect
- Ensure that the Confluent for Kubernetes operator is installed on your Kubernetes cluster. See the [Confluent for Kubernetes Quickstart](https://docs.confluent.io/operator/current/co-quickstart.html) for more information.
- Create a new YAML file for Kafka Connect, as shown below.

    - The `build` section references the URL and SHA512 checksum for the modified JDBC connector you previously created.
      - If you prefer, you may keep the values for `archivePath` and `checksum` listed below to use the customized zip file prepared for this example (assuming the file is still hosted at the time you are reading this). 
    - The `dependencies` section contains the bootstrap URL (as found in _Cluster Settings_ at https://confluent.cloud/), the secret containing the API key, and TLS settings for Kafka Connect to communicate with Confluent Cloud using the pre-installed TLS certificates in the image.
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

- Apply the manifest to create the Connect cluster:  
  `kubectl apply -f connect.yaml`

## Create YAML file for the Snowflake JDBC Connector
- At this point, you have a Kafka Connect cluster running and connected to your Confluent Cloud cluster. It has the Snowflake JDBC Source Connector plugin installed, so a connector of that type can be deployed.
- Now we simply need to configure the Snowflake JDBC Source connector, carefully referencing all of the relevant Snowflake configurations (e.g., `connect.url`, `warehouse`, etc.). If needed, reference Snowflake's [JDBC Driver Connection Parameter Reference](https://docs.snowflake.com/developer-guide/jdbc/jdbc-parameters) page for more information.
- Create a new YAML file for the connector, as shown below.
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
- Apply the manifest to create the connector:  
  `kubectl apply -f connector.yaml`

## _Optional_ - configure Confluent Control Center 
- You can configure Confluent Control Center so that you can monitor your Connect cluster (as well as other Confluent components, including Confluent Cloud Kafka clusters) using a self-hosted Web UI.
- Create a new YAML file for Control Center, as shown below.
  - The `kafka` dependency can be removed if you only need Control Center to manage the Connect cluster. In this example, it is included and uses the same API key/secret that was used by the Connect cluster.
  - `externalAccess` is only needed if you want to use a load balancer to access Control Center externally. To get started, you may remove this whole section and instead just run `kubectl port-forward controlcenter-0 9021:9021`, which will allow you to open the Control Center web page in your local browser (or from wherever the command is ran).
    - `your-domain.com` should be replaced with the domain where your Kubernetes cluster is running. You may be able to enter any value here and still access Control Center using the external IP address assigned to the Control Center service. To retrieve this IP address, you can execute `kubectl get service controlcenter-bootstrap-lb`
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
All of the following commands assume that your kubectl context is set to the confluent namespace (or whichever namespace resources are deployed to). To permanently set the namespace for all subsequent kubectl commands in your context, you can run:  
`kubectl config set-context --current --namespace=confluent`
- See the status of all Confluent [custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/):
  - `kubectl get confluent`
- Describe the Connect, Connector, or ControlCenter custom resources:
  - `kubectl describe connect connect` (the name of the Connect resource _is_ connect)
  - `kubectl describe connector snowflake-jdbc-source`
  - `kubectl describe controlcenter controlcenter` (the name of the ControlCenter resource _is_ controlcenter)
- Show the Confluent for Kubernetes pod logs:
  - `kubectl logs confluent-operator-xxxxxxxxxx-xxxxx`
- Show the Connect pod logs:
  - `kubectl logs connect-0`
- Show the ControlCenter pod logs:
  - `kubectl logs controlcenter-0`  
- Show the Connect init container's pod logs (may be useful if connector plugin is not available):
  - `kubectl logs connect-0 -c config-init-container`
- Show all events in the namespace, sorted by timestamp:
  - `kubectl get events --sort-by=.metadata.creationTimestamp`