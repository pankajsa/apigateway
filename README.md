# apigateway
How to setup Solace PubSub+ as API Micro-Gateway.

PubSub+ can be configured as a API gateway in a number of ways. The three patterns are below in increasing order of benefits.

This repository explains the setup and how it can be implemented using simple curl commands. 

- Proxy to Backend μservices using HTTP(s)
- Backend μservices using Messaging
- API G/W in DMZ and Backend μservices using Messaging (preferred)


## Proxy to Backend μservices using HTTP(s)


### Running Solace PubSub+ as Docker Container

Get the PubSub+ image
```shell
docker run -d -p 8080:8080 -p 55555:55555  -p 8000:8000 -p 9000:9000 -p 9443:9443  --shm-size=2g --env 'username_admin_globalaccesslevel=admin' --env 'username_admin_password=admin' --name=solace solace/solace-pubsub-standard:9.3.0.22
        
```

It will take about 3-4 minutes for PubSub+ to start. You can tail the docker logs using the below command. The container starts up after you see 'Running pre-startup checks: [  OK  ]' in the log.

```
docker logs -f solace
```
Use the web admin at http://localhost:8080 (username is _admin_ and the password is _admin_)

### Configure the Broker
Switch the broker to _gateway_ mode
```shell
curl -X PATCH  -u admin:admin http://localhost:8080/SEMP/v2/config/msgVpns/default \
    -H "content-type: application/json" -d \
    '{"serviceRestMode":"gateway"}'
```
Create the queue **API_RQ** on the broker that will get a copy of all the API requests.
```shell
curl -X POST -u admin:admin http://localhost:8080/SEMP/v2/config/msgVpns/default/queues \
    -H "content-type: application/json" -d \
    '{"msgVpnName" :"default","queueName":"API_RQ", "permission":"consume", "ingressEnabled":true, "egressEnabled":true, "respectTtlEnabled":true, "maxTtl":10}'
```
Create the queue **API_RQ** on the broker that will get a copy of all the API requests.
```shell
curl -X POST -u admin:admin http://localhost:8080/SEMP/v2/config/msgVpns/default/queues \
    -H "content-type: application/json" -d \
    '{"msgVpnName" :"default","queueName":"API_RQ", "permission":"consume", "ingressEnabled":true, "egressEnabled":true, "respectTtlEnabled":true, "maxTtl":10}'
```
Setup the subscriptions in the queue for API requests. For this initial setup we are using a single queue, but for a scenario where the REST microservices are available at different endpoints you should create multiple qeueus. This is explained later.
```shell
curl -X POST -u admin:admin http://localhost:8080/SEMP/v2/config/msgVpns/default/queues/API_RQ/subscriptions \
    -H "content-type: application/json" -d \
    '{"subscriptionTopic":"GET/>"}'
curl -X POST -u admin:admin http://localhost:8080/SEMP/v2/config/msgVpns/default/queues/API_RQ/subscriptions \
    -H "content-type: application/json" -d \
    '{"subscriptionTopic":"POST/>"}'
```

Create a REST Delivery Point.
```shell
curl -X POST -u admin:admin http://localhost:8080/SEMP/v2/config/msgVpns/default/restDeliveryPoints \
    -H "content-type: application/json" -d \
    '{"restDeliveryPointName":"myRdp1", "enabled":true}'
```
Bind the queue that gets the REST requests to RDP
```shell
curl -X POST -u admin:admin http://localhost:8080/SEMP/v2/config/msgVpns/default/restDeliveryPoints/myRdp1/queueBindings \
    -H "content-type: application/json" -d \
    '{"queueBindingName":"API_RQ"}'
```

Setup the URL for the backend microservice
```shell
export backend_host=httpbin.org
export backend_port=80
curl -v -X POST -u admin:admin http://localhost:8080/SEMP/v2/config/msgVpns/default/restDeliveryPoints/myRdp1/restConsumers \
    -H "content-type: application/json" -d \
    '{"restConsumerName":"myRESTBackend1" ,"remoteHost":"'$backend_host'","remotePort":'$backend_port', "tlsEnabled":false, "enabled":true}'
```

Verify that you are able to now send the REST request
```shell
curl http://localhost:9000/get
```


#Rest Consumer target URL http://backend-server:port/get/AA
curl -X POST -u admin:admin $vmr_ip:8080/SEMP/v2/config/msgVpns/default/restDeliveryPoints/aa/restConsumers  -H "content-type: application/json" -d '{"msgVpnName":"default","restConsumerName":"backend_aa" ,"restDeliveryPointName":"aa" ,"remoteHost":"'"$backend_ip"'","remotePort":1981, "tlsEnabled":false, "enabled":true}'




## Backend μservices using Messaging
## API G/W in DMZ and Backend μservices using Messaging (preferred)


