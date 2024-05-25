# hudi-streamer-pulsar
hudi-streamer-pulsar


![1716560911929](https://github.com/soumilshah1995/hudi-streamer-pulsar/assets/39345855/8cf2ba22-ba47-46aa-a61b-e98aff38b77b)

# Steps 

# Step 1: Spin up pulsar
```
docker run -it \
-p 6650:6650 \
-p 8080:8080 \
--mount source=pulsardata,target=/pulsar/data \
--mount source=pulsarconf,target=/pulsar/conf \
apachepulsar/pulsar:3.2.3 \
bin/pulsar standalone
```

# Step 1:Spin up MINIO
```
version: '3'

services:
  minio:
    image: minio/minio
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
      - MINIO_DOMAIN=minio
    networks:
      default:
        aliases:
          - warehouse.minio
    ports:
      - 9001:9001
      - 9000:9000
    command: [ "server", "/data", "--console-address", ":9001" ]

  mc:
    depends_on:
      - minio
    image: minio/mc
    environment:
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
    entrypoint: >
      /bin/sh -c "
      until (/usr/bin/mc config host add minio http://minio:9000 admin password) do echo '...waiting...' && sleep 1; done;
      /usr/bin/mc rm -r --force minio/warehouse;
      /usr/bin/mc mb minio/warehouse;
      /usr/bin/mc policy set public minio/warehouse;
      tail -f /dev/null
      "



```

# Step 3: Publish sample messages 
```

import datetime
import pulsar
import fastavro
print("ok")
from puavro import DictAVRO, DictAvroSchema

PULSAR_SERVICE_URL = "pulsar://localhost:6650"
TOPIC = "try"


class Segment(DictAVRO):
    SCHEMA = fastavro.schema.parse_schema({
        "type": "record",
        "name": "Segment",
        "namespace": "try",
        "fields": [
            {"name": "id", "type": "long"},
            {"name": "name", "type": "string"},
            {
                "name": "when",
                "type": {
                    "type": "long",
                    "logicalType": "timestamp-millis"
                }
            },
            {
                "name": "direction",
                "type": {
                    "type": "enum",
                    "name": "CardinalDirection",
                    "symbols": ["north", "south", "east", "west"]
                }
            },
            {"name": "length", "type": ["null", "long"]}
        ]
    })


# Create Pulsar client
pulsar_client = pulsar.Client(PULSAR_SERVICE_URL)
# Create a producer with the schema
producer = pulsar_client.create_producer(
    topic=TOPIC,
    schema=DictAvroSchema(Segment)
)

try:
    # Create a Segment instance
    segment = Segment(
        id=99,
        name="some name",
        when=int(datetime.datetime.utcnow().replace(tzinfo=datetime.timezone.utc).timestamp() * 1000),
        direction="north",
        length=12345,
    )
    # Send the message
    producer.send(segment)


    print("done")
finally:
    # Close the producer and client
    producer.close()
    pulsar_client.close()


```

# Step 4: Delta Streamer 



spark-config.properties

```

spark.serializer=org.apache.spark.serializer.KryoSerializer
spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog
spark.sql.hive.convertMetastoreParquet=false

spark.hadoop.fs.s3a.access.key=admin
spark.hadoop.fs.s3a.secret.key=password
spark.hadoop.fs.s3a.endpoint=http://127.0.0.1:9000
spark.hadoop.fs.s3a.path.style.access=true
fs.s3a.signing-algorithm=S3SignerType

```


gh_schema.avsc

```
{
  "type": "record",
  "name": "Segment",
  "namespace": "try",
  "fields": [
    {
      "name": "id",
      "type": "long"
    },
    {
      "name": "name",
      "type": "string"
    },
    {
      "name": "when",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      }
    },
    {
      "name": "direction",
      "type": {
        "type": "enum",
        "name": "CardinalDirection",
        "symbols": ["north", "south", "east", "west"]
      }
    },
    {
      "name": "length",
      "type": ["null", "long"]
    }
  ]
}

```


# Streamer
```
spark-submit \
    --class org.apache.hudi.utilities.streamer.HoodieStreamer \
    --packages 'org.apache.hudi:hudi-spark3.4-bundle_2.12:0.14.0,io.streamnative.connectors:pulsar-spark-connector_2.12:3.1.1.4,org.slf4j:slf4j-api:1.7.32' \
    --repositories https://repo.maven.apache.org/maven2 \
    --properties-file spark-config.properties \
    --master 'local[*]' \
    --executor-memory 1g \
    /Users/soumilshah/IdeaProjects/SparkProject/apache-hudi-delta-streamer-labs/E5/jar/hudi-utilities-slim-bundle_2.12-0.14.0.jar \
    --table-type COPY_ON_WRITE \
    --op UPSERT \
    --continuous \
    --source-limit 4000000 \
    --min-sync-interval-seconds 20 \
    --source-ordering-field ts \
    --source-class org.apache.hudi.utilities.sources.PulsarSource \
    --target-base-path file:///Users/soumilshah/IdeaProjects/SparkProject/ApachePulsar/hudi \
    --target-table orders \
    --hoodie-conf hoodie.datasource.write.keygenerator.class=org.apache.hudi.keygen.SimpleKeyGenerator \
    --hoodie-conf hoodie.datasource.write.recordkey.field=order_id \
    --hoodie-conf hoodie.datasource.write.partitionpath.field=order_date \
    --hoodie-conf hoodie.datasource.write.precombine.field=ts \
    --hoodie-conf hoodie.deltastreamer.source.pulsar.topic=orders \
    --hoodie-conf hoodie.deltastreamer.source.pulsar.offset.autoResetStrategy=LATEST \
    --hoodie-conf hoodie.deltastreamer.source.pulsar.endpoint.service.url=pulsar://localhost:6650 \
    --hoodie-conf hoodie.deltastreamer.source.pulsar.endpoint.admin.url=http://localhost:8080


```
