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

You should get a response similar to which means that request has gone from your client all the way to httpbin.org and back.
```shell
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "Solace-Delivery-Mode": "Non-Persistent",
    "Solace-Message-Id": "ID:Solace-704ca5d897cea4f2",
    "Solace-Reply-Wait-Time-In-Ms": "FOREVER",
    "Solace-Time-To-Live-In-Ms": "30000",
    "User-Agent": "curl/7.64.1"
  },
  "origin": "103.252.202.112, 103.252.202.112",
  "url": "https://httpbin.org/get"
}
```

### So what's the big deal here
All API requests are now event enabled. You can have a concurrent messaging application listening to topic "GET/>" or "POST/>" to get a copy of the messages and do asynchonous processing e.g. analytics, machine learning etc.

### Multiple μservices
Instead of having a single queue you can have multiple queues. For e.g. API_CUST_RQ can subscribe to **GET/customer/>** to handle the API requests for **customer** as the bounded context, and another queue API_PAY_RQ can subscribe to **POST/pay/>** to handle the requests for **payment** domain.


## Backend μservices using Messaging
## API G/W in DMZ and Backend μservices using Messaging (preferred)


