# Initial Setup

## Configuration

The assumption is that Kong will be accessible via the api.kong.lan hostname. Ensure this resolves to the correct IP for where you will be running Kong.

The docker-compose file expects to find the SSL certifcate pairs in the `./ssl-certs` and `./ssl-certs/hybrid` directories in this repository; these directories are mapped via docker volumes in the docker-compose file for Kong to access the certificates. There are two sets of certificates required, the first for HTTPS access to Kong Manager and the second is for Control Plane/Data Plane communication.

1) Create the SSL certificates for the api.kong.lan hostname [here](ssl-certs/README.md)

2) Create the hybrid CP/DP certs [here](ssl-certs/hybrid/README.md)

## Start containers

Set and env var for the license;

~~~
export KONG_LICENSE_DATA=`cat ./license.json`;
~~~

Then start the utility services & kong containers

~~~
docker-compose up -d
~~~

This will start Kong EE, Postgres, Keycloak, an LDAP (AD) server, an HAProxy server and a Locust load testing server. 

## Authentication

By default, ldap-auth is enabled and you can login with kong_admin/K1ngK0ng

You can look at the LDAP tree by searching as below;

~~~
ldapsearch -H "ldap://0.0.0.0:389" -D "cn=Administrator,cn=users,dc=ldap,dc=kong,dc=com" -w "Passw0rd" -b "dc=ldap,dc=kong,dc=com" "(sAMAccountName=kong_admin)"
~~~

## Test kong is working by making an Admin API request

It is necessary to pass the CA certificate with the request to allow curl to verify the certs (or use -k which is not recommended);

~~~
curl --cacert ./ssl-certs/rootCA.pem -H "kong-admin-token: password" https://api.kong.lan:48444/default/kong
~~~

## Default endpoints for HAProxy healthcheck & httpbin

A deck container is used to populate the endpoints used for the ha-proxy health check as well as creating a route/service for the local httpbin

## Test a proxy

Test the default API via the HAProxy (the Kong proxy ports are not exposed externally so access in *ONLY* via HaProxy);

~~~
$ curl http://api.kong.lan/httpbin/anything
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "httpbin-1",
    "User-Agent": "curl/7.64.0",
    "X-Forwarded-Host": "api.kong.lan",
    "X-Forwarded-Prefix": "/httpbin"
  },
  "json": null,
  "method": "GET",
  "origin": "172.28.0.1, 172.28.0.13",
  "url": "http://api.kong.lan/anything"
}

$ curl -s --cacert ./ssl-certs/rootCA.pem --http2 https://api.kong.lan/httpbin/anything
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "httpbin-1",
    "User-Agent": "curl/7.64.0",
    "X-Forwarded-Host": "api.kong.lan",
    "X-Forwarded-Prefix": "/httpbin"
  },
  "json": null,
  "method": "GET",
  "origin": "172.28.0.13",
  "url": "https://api.kong.lan/anything"
}
~~~

There is also a rate limited example, secured with key-auth and two consumers

~~~
$ curl http://api.kong.lan/limit-httpbin/anything?apikey=abc
{
  "args": {
    "apikey": "abc"
  },
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "kongpose_httpbin_2",
    "User-Agent": "curl/7.72.0",
    "X-Consumer-Id": "ef320150-1b41-4af7-b567-bce8e093c6a6",
    "X-Consumer-Username": "consA",
    "X-Credential-Identifier": "c507fb51-1af9-4ca9-b8bf-5a520c83be58",
    "X-Forwarded-Host": "api.kong.lan",
    "X-Forwarded-Path": "/limit-httpbin/anything",
    "X-Forwarded-Prefix": "/limit-httpbin"
  },
  "json": null,
  "method": "GET",
  "origin": "172.26.0.1, 172.26.0.23",
  "url": "http://api.kong.lan/anything?apikey=abc"
}
~~~

and

~~~
$ curl http://api.kong.lan/limit-httpbin/anything?apikey=123
{
  "args": {
    "apikey": "123"
  },
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "kongpose_httpbin_2",
    "User-Agent": "curl/7.72.0",
    "X-Consumer-Id": "36fb326f-e011-4d2b-8acc-dd9638615d8b",
    "X-Consumer-Username": "consB",
    "X-Credential-Identifier": "a01449f0-bf58-44ec-8a5c-1f38deabcb93",
    "X-Forwarded-Host": "api.kong.lan",
    "X-Forwarded-Path": "/limit-httpbin/anything",
    "X-Forwarded-Prefix": "/limit-httpbin"
  },
  "json": null,
  "method": "GET",
  "origin": "172.26.0.1, 172.26.0.23",
  "url": "http://api.kong.lan/anything?apikey=123"
}
~~~

Send a few requests, get a 429 response and take a look in [redis](README.md#redis) ;-)

# Websocket demo

The echo-server can be used for websocket echo functionality. There is a default route setup on `/echo` that can be connected to with `websocat` or a websocket client of your choice. This server will echo back any data sent to it;

```
$ websocat ws://api.kong.lan/echo
Request served by d5dd815f8b0d
this is an echo
this is an echo
```

Compared to an HTTP request to this server;

```
$ curl http://api.kong.lan/echo
Request served by d5dd815f8b0d

HTTP/1.1 GET /

Host: echo-server:8080
Connection: keep-alive
X-Forwarded-Proto: http
X-Forwarded-Path: /echo
User-Agent: curl/7.74.0
X-Real-Ip: 192.168.80.1
Accept: */*
X-Forwarded-For: 192.168.80.1, 192.168.80.22
X-Forwarded-Host: api.kong.lan
X-Forwarded-Port: 48000
X-Forwarded-Prefix: /echo
```

# GRPC Example

## TLS

```
$ grpcurl -cacert ./ssl-certs/rootCA.pem -v -H 'kong-debug: 1' -d '{"greeting": "Kong 1.3!"}'  api.kong.lan:443 hello.HelloService.SayHello

Resolved method descriptor:
rpc SayHello ( .hello.HelloRequest ) returns ( .hello.HelloResponse );

Request metadata to send:
kong-debug: 1

Response headers received:
content-type: application/grpc
date: Wed, 13 Jan 2021 12:20:23 GMT
kong-route-id: 126677da-3ebb-4d72-b52e-b2cb31309383
kong-route-name: local-grpc-sayHello
kong-service-id: 70ff022b-b275-4d49-8f5a-a7b16ed8c78e
kong-service-name: local-grpc-server
server: openresty
via: kong/2.2.1.0-enterprise-edition
x-kong-proxy-latency: 1
x-kong-upstream-latency: 2

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response trailers received:
(empty)
Sent 1 request and received 1 response
```

```
$ grpcurl -cacert ./ssl-certs/rootCA.pem -v -H 'kong-debug: 1' -d '{"greeting": "Kong 1.3!"}'  api.kong.lan:443 hello.HelloService.LotsOfReplies

Resolved method descriptor:
rpc LotsOfReplies ( .hello.HelloRequest ) returns ( stream .hello.HelloResponse );

Request metadata to send:
kong-debug: 1

Response headers received:
content-type: application/grpc
date: Wed, 13 Jan 2021 12:19:17 GMT
kong-route-id: 1307e24e-0c3d-40bf-875a-d0ff61861c8d
kong-route-name: local-grpc-lotsOfReplies
kong-service-id: 70ff022b-b275-4d49-8f5a-a7b16ed8c78e
kong-service-name: local-grpc-server
server: openresty
via: kong/2.2.1.0-enterprise-edition
x-kong-proxy-latency: 1
x-kong-upstream-latency: 1

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response contents:
{
  "reply": "hello Kong 1.3!"
}

Response trailers received:
(empty)
Sent 1 request and received 10 responses
```

# Keycloak:

URL: http://api.kong.lan:8080

Username: admin

Password: password

# HAProxy

https://www.haproxy.com/blog/dynamic-configuration-haproxy-runtime-api/

https://www.haproxy.com/documentation/hapee/1-9r1/onepage/management/#9.3

To use the haproxy API;

~~~
echo "help" | socat stdio tcp4-connect:api.kong.lan:9999
~~~

## Gracefully remove server from rotation
~~~
echo "set server kong_http_proxy_nodes/kongpose_kong-dp_3 state drain" | socat stdio tcp4-connect:api.kong.lan:9999
echo "set server kong_https_proxy_nodes/kongpose_kong-dp_3 state drain" | socat stdio tcp4-connect:api.kong.lan:9999
~~~

## Disable healthchecks for server
~~~
echo "disable health kong_http_proxy_nodes/kongpose_kong-dp_3" | socat stdio tcp4-connect:api.kong.lan:9999
echo "disable health kong_https_proxy_nodes/kongpose_kong-dp_3" | socat stdio tcp4-connect:api.kong.lan:9999
~~~

## Set weight for server
~~~
echo "set server kong_http_proxy_nodes/kongpose_kong-dp_3 weight 50%" | socat stdio tcp4-connect:api.kong.lan:9999
echo "set server kong_https_proxy_nodes/kongpose_kong-dp_3 weight 50%" | socat stdio tcp4-connect:api.kong.lan:9999
~~~

## Enable healthchecks for server
~~~
echo "enable health kong_http_proxy_nodes/kongpose_kong-dp_3" | socat stdio tcp4-connect:api.kong.lan:9999
echo "enable health kong_https_proxy_nodes/kongpose_kong-dp_3" | socat stdio tcp4-connect:api.kong.lan:9999
~~~

## Add server back to rotation
~~~
echo "set server kong_http_proxy_nodes/kongpose_kong-dp_3 state ready" | socat stdio tcp4-connect:api.kong.lan:9999
echo "set server kong_https_proxy_nodes/kongpose_kong-dp_3 state ready" | socat stdio tcp4-connect:api.kong.lan:9999
~~~

To view some HAProxy stats, look here;

http://api.kong.lan:8404/stats

# Locust

For some loadtesting ability, checkout locust; https://locust.io/

Locust is available in this case testing environment at http://api.kong.lan:8089/ and a simple test will hit the `httpbin/status/200` and `httpbin/status/503` endpoints in a 100-to-1 split. Edit the [locustfile.py](locust/locustfile.py) file to make the test more interesting.

# SMTP Server

Add a fake-smtp-server service. View the UI at http://api.kong.lan:1080/

# Redis

Need to see what is stored in redis? Try redis-commander at http://api.kong.lan:8081

# Graylog

For Syslog/TCP/UDP log plugins, use Graylog. The default username/password to login is admin/admin

http://api.kong.lan:9000/

Two example inputs (System/Inputs) have been created via the "content packs" JSON file to listen on port 5555 for TCP and UDP.

# Zipkin

View traces here;

http://api.kong.lan:9411/
