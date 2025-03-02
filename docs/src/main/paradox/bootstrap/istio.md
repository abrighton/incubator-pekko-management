# Istio

[Istio](https://istio.io/) is a service mesh that handles many of the concerns of service to service communication for you, such as routing, load balancing, authentication, authorization, and observability. In certain use cases, it can complement an Pekko cluster well. Typically, Pekko cluster communication is used for multiple nodes of the same service to communicate with each other, to achieve things such as sharding, replication and peer to peer communication, while a service mesh might be used for nodes of different services to communicate with each other, through REST and gRPC APIs.

To bootstrap a Pekko cluster in Istio, Istio must be configured to allow Pekko cluster communication to bypass the Istio sidecar proxy. Istio's routing design is made such that services don't need to be aware of each others location, they just communicate with the proxy, and the mesh figures out how to route and secure the communication. However, Pekko cluster communication is fundamentally location aware, in order to, for example, route messages to sharded actors. Hence a service mesh is not a suitable communication medium for cluster traffic, so it needs to be bypassed.

It is important to be aware that since Istio's proxy is bypassed, the Pekko cluster communication will not be secured by Istio using TLS. If you wish to secure your cluster communication, you will need to configure [Akka remoting with mTLS](https://pekko.apache.org/docs/pekko/current/remoting-artery.html#remote-security) yourself.

Booting a Pekko cluster in Istio requires a minimum Istio version of 1.2.0, as it requires the outbound port exclusions feature that was added in there. It also requires using the @ref[Kubernetes API](kubernetes-api.md) contact point discovery method to be used. The instructions below are for the additional configuration necessary to ensure an Akka cluster can be bootstrapped in Istio.

## Allowing outbound communication

By default, Istio redirects all outbound communication to its proxy. To prevent it from doing this for Pekko cluster communication, both the remoting and management ports need to be excluded from redirection. This can be done using the `traffic.sidecar.istio.io/excludeOutboundPorts` annotation in the deployment pod template. If your remoting port is 7355, and management port is 7626, this can be done like so:

```yaml
annotations:
  traffic.sidecar.istio.io/excludeOutboundPorts: "7355,7626"
```

## Allowing inbound communication

Inbound connections to Pekko management and remoting also need to bypass the sidecar proxy. By default, Istio will redirect all incoming traffic to the ports listed in the containers port specification to the sidecar proxy. Hence, there are two ways to ensure that the Pekko management and remoting traffic bypasses the proxy, either explicitly configure the incoming ports to redirect, or don't list the Pekko management and remoting ports in the container's ports specification.

The inbound ports to redirect can be configured using the `traffic.sidecar.istio.io/includeInboundPorts` annotation. If your service offers a REST endpoint on port 8080, then you might configure that like so:

```yaml
annotations:
  traffic.sidecar.istio.io/includeInboundPorts: "8080"
```

If you offer any other services on other ports, they can be added as a comma separated list. The important thing to ensure is that your remoting and management ports are not listed in that list.

## Example

Here is an example deployment spec for an Pekko cluster deployed to Istio, which uses the explicit include inbound ports annotation to ensure that incoming remoting and management traffic isn't redirected through the proxy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
      annotations:
        traffic.sidecar.istio.io/includeInboundPorts: "8080"
        traffic.sidecar.istio.io/excludeOutboundPorts: "7355,7626"
    spec:
      containers:
      - name: my-service
        image: "my-service:latest"

        ports:
        - containerPort: 8080
          name: http
        - containerPort: 7355
          name: remoting
        - containerPort: 7626
          name: management

        readinessProbe:
          httpGet:
            path: "/ready"
            port: management
        livenessProbe:
          httpGet:
            path: "/alive"
            port: management
```