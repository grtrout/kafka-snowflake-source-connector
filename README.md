# Create and run a Snowflake JDBC Source Connector for Confluent Cloud

## Download the the JDBC Source Connector and the Snowflake JDBC driver
- Download the Snowflake JDBC driver.  
https://repo1.maven.org/maven2/net/snowflake/snowflake-jdbc/3.13.16/snowflake-jdbc-3.13.16.jar

- Download the Confluent JDBC Connector from [Confluent Hub](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc).  

## Add the Snowflake JDBC driver to the JDBC Source Connector
- Extract the Confluent JDBC Connector zip file and navigate to the lib folder.

- Copy the Snowflake JDBC driver JAR (`snowflake-jdbc-3.13.16.jar`) and paste it into this lib folder.

- Compress the entire folder as a zip - just as it was before you extracted it before. You now have a JDBC Connector plugin with the added Snowflake JDBC driver.

- Note the SHA512 checksum of this newly created file (you will need this later).
  - `shasum -a 512 '<your new zip file>'`

## Upload the updated JDBC Source Connector
- Upload this new zip file to a location that can be reached from the Kubernetes cluster where Kafka Connect is deployed.

- ? added to the plugin url list

## Create and use a Confluent Cloud API Key
- Log on to Confluent Cloud at https://confluent.cloud/, navigate to your cluster, and create an API key.

- Create a file named `plain.txt` with the following contents:
```
username=<API Key>
password=<API Secret>
```

- Create a Kubernetes secret for the API key, referencing this file (this secret, named `ccloud-credentials` will be referenced later).  
`kubectl create secret generic ccloud-credentials --from-file=plain.txt`

## Create YAML file for Kafka Connect
- Ensure that the Confluent for Kubernetes operator is installed on your Kubernetes cluster. See the [Confluent for Kubernetes Quickstart](https://docs.confluent.io/operator/current/co-quickstart.html) for more information.
- Create a new YAML file for Kafka Connect, as shown below.

    - The `build` section references the URL and SHA512 checksum for the modified JDBC connector you previously created.
    - The `dependencies` section contains the bootstrap URL (as found in Cluster Settings at https://confluent.cloud/), the secret containing the API key, and TLS settings for Kafka COnnect to use the pre-installed TLS certificates in the image for Kafka Broker communications.
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
            archivePath: "https://bucket-name.s3.us-east-2.amazonaws.com/confluentinc-kafka-connect-jdbc-snowflake-10.7.3.zip"
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
   - `kubectl apply -f connect.yaml`

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
   - `kubectl apply -f connector.yaml`
