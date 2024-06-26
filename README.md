# mqtt-to-s3

Store MQTT messages in S3 - part of the course [Introduction to Kubernetes on Edge with K3s](https://www.edx.org/course/introduction-to-kubernetes-on-edge-with-k3s)

This sample uses an [OpenFaaS MQTT-connector](https://github.com/openfaas/mqtt-connector) along with a Python function to receive JSON messages from MQTT and to store them in an S3 bucket.

Follow the instruction below by running the commands in your terminal to create infrastructure that receives ingestion of IoT sensor data from edge devices. It is a starting point, and easy to modify for your own uses.

## Installation

Install [arkade](https://get-arkade.dev), the open-source Kubernetes marketplace:

```bash
# Have arkade move itself into /usr/local/bin/
curl -sLS https://dl.get-arkade.dev | sudo sh
```

Install [Minio](https://min.io/) and [OpenFaaS](https://www.openfaas.com/) to your Kubernetes cluster:

```bash
arkade install openfaas

export PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo) && export PATH=$PATH:$HOME/.arkade/bin/

kubectl rollout status -n openfaas deploy/gateway 

#Press Enter to push the port-forwarding process to the background after running the following command.
kubectl port-forward -n openfaas svc/gateway 8080:8080 > openfaassvc.log 2>&1 & 

#Install the CLI for OpenFaaS:
arkade get faas-cli
export PATH=$PATH:$HOME/.arkade/bin/

#Log in to OpenFaas
echo -n $PASSWORD | faas-cli login --username admin --password-stdin


arkade install minio
# Log in with the following command.
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=minio" -o jsonpath="{.items[0].metadata.name}")

```

Get CLIs for the Minio client:

```bash
arkade get mc
# Add arkade binary directory to your PATH variable
export PATH=$PATH:$HOME/.arkade/bin/
```

You must have the latest version of faas-cli for this step, check it with `faas-cli version`

Then set up secret for Minio:

```bash
export ACCESSKEY=$(kubectl get secret -n default minio -o jsonpath="{.data.root-user}" | base64 --decode; echo)
export SECRETKEY=$(kubectl get secret -n default minio -o jsonpath="{.data.root-password}" | base64 --decode; echo)
echo -n $SECRETKEY > ./secret-key.txt
echo -n $ACCESSKEY > ./access-key.txt

faas-cli secret create secret-key --from-file ./secret-key.txt --trim
faas-cli secret create access-key --from-file ./access-key.txt --trim
```

Forward ports for the minio service and setup `mc` to access Minio:

```bash
kubectl port-forward svc/minio 9000:9000 > ~/miniosvc.log 2>&1 &

mc alias set minio http://127.0.0.1:9000 $(cat access-key.txt) $(cat secret-key.txt)

Added `minio` successfully.
```

Then make a bucket for the sensor data:

```bash
mc mb minio/sensor-data

# Show the bucket exists:

mc ls minio
[2021-07-17 12:34:06 BST]     0B sensor-data/

# Show that it's empty, and found
mc ls minio/sensor-data
```

Then install the mqtt-connector, which will invoke your functions:

```bash
# Use a unique client ID
# client ID conflicts will cause connect events to fire over and over
# https://github.com/eclipse/paho.mqtt.golang#common-problems
CLIENT_ID=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)

arkade install mqtt-connector \
  --topics openfaas-sensor-data \
  --broker-host tcp://test.mosquitto.org:1883 \
  --client-id $CLIENT_ID

kubectl logs deploy/mqtt-connector -n openfaas -f
```

Next, deploy the functions:

```bash
# Change directory to this cloned repo

# get the template this function depends on
faas-cli template store pull python3-flask

# deploy this function
faas-cli deploy
```
Install pip3 with your package manager if it isn't already installed. On Ubuntu, the following command will install pip3:
```
$ sudo apt install python3-pip
```
## Testing the functions

Then publish an event to the "openfaas-sensor-data" topic on the iot.eclipse.org test server.

You can run the [sender/send.py](sender/send.py) file to publish a message to the topic. Our function will store the message in an S3 bucket.

First run:

```bash
pip3 install paho-mqtt
```


Then run `sender/send.py`

```bash
$ python3 sender/send.py '{"sensor_id": 1, "temperature_c": 50}'

Connecting to test.mosquitto.org:1883
Connected with result code 0
Message "{"sensor_id": 1, "temperature_c": 53}" published to "openfaas-sensor-data"

$ python3 sender/send.py '{"sensor_id": 1, "temperature_c": 53}'
```

Check the logs of the function to see that it was invoked:

```bash
faas-cli logs mqtt-s3

2021-03-14T13:49:13Z 2021/03/14 13:49:13 POST / - 200 OK - ContentLength: 2
2021-03-14T13:50:01Z 2021/03/14 13:50:01 POST / - 200 OK - ContentLength: 2
```

Now check the contents of your S3 bucket with `mc`:

```
mc ls minio/sensor-data

[2021-03-14 09:50:01 EDT]    35B 23c7cb33-246c-4173-af3e-5ce2898aa564.json
[2021-03-14 09:49:13 EDT]    35B 77fb7012-e893-40d6-b23f-477a143bcb5f.json
```

You should see a number of .json files created for each message you create with the sender.

You can also retrieve the file to view it with:

```bash
mc cp minio/sensor-data/a3832ae7-3381-41b7-a0f0-c53533728a1f.json .
```

Now view the result stored in the file:

```bash
cat a3832ae7-3381-41b7-a0f0-c53533728a1f.json

{"sensor_id": 1, "temperature_c": 53}
```


