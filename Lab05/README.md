# Lab05 - Kubernetes Network Policy and Calico 

NetworkPolicy resources use labels to select pods and define rules which specify what traffic is allowed to the selected pods. By default, pods accept traffic from any source. Once there is any NetworkPolicy in a namespace selecting a particular pod, that pod will reject any connections not allowed by any NetworkPolicy. 

Network policies do not conflict; they are additive. If any policy or policies select a pod, the pod is restricted to what is allowed by the union of those policies’ ingress/egress rules. Thus, order of evaluation does not affect the policy result.

There are four kinds of selectors in an ingress `from` section or egress `to` section:
- podSelector,
- namespaceSelector,
- podSelector and namespaceSelector,
- ipBlock for IP CIDR ranges.

For example,

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 5978
```

Every IBM Cloud Kubernetes Service cluster is set up with a network plug-in called Calico. Default network policies are set up to secure the public network interface of every worker node in the cluster.

You can use Kubernetes and Calico to create network policies for a cluster. With Kubernetes network policies, you can specify the network traffic that you want to allow or block to and from a pod within a cluster. To set more advanced network policies such as blocking inbound (ingress) traffic to network load balancer (NLB) services, use Calico network policies.

When a Kubernetes network policy is applied, it is automatically converted into a Calico network policy so that Calico can apply it as an Iptables rule. Both incoming and outgoing network traffic can be allowed or blocked based on protocol, port, and source or destination IP addresses. Traffic can also be filtered based on pod and namespace labels.

Calico network policies are a superset of the Kubernetes network policies and are applied by using calicoctl commands. Calico policies add the following features:
- Allow or block network traffic on specific network interfaces regardless of the Kubernetes pod source or destination IP address or CIDR.
- Allow or block network traffic for pods across namespaces.
- Block inbound traffic to Kubernetes LoadBalancer or NodePort services.

Calico enforces these policies, including any Kubernetes network policies that are automatically converted to Calico policies, by setting up Linux Iptables rules on the Kubernetes worker nodes. Iptables rules serve as a firewall for the worker node to define the characteristics that the network traffic must meet to be forwarded to the targeted resource.

# Adopt a zero trust network model

Adopting a zero trust network model is best practice for securing workloads and hosts in your cloud-native strategy.

## Create helloworld Proxy

Deploy the `helloworld` proxy,

```
$ kubectl create -f helloworld-proxy-deployment.yaml
$ kubectl create -f helloworld-proxy-service-loadbalancer.yaml
```

Get the proxy service details and test the proxy,

```
$ kubectl get svc helloworld-proxy
NAME    TYPE    CLUSTER-IP    EXTERNAL-IP    PORT(S)    AGE
helloworld-proxy    LoadBalancer    172.21.78.158    169.48.67.166    8080:32333/TCP    61s
```

To test use the External-IP and the NodePort, e.g. 169.48.67.166:32333, and send the proxy the Kubernetes service name and target port,

```
$ PROXY_IP=169.48.67.166
$ PORT_P=32333
$ curl -L -X POST "http://$PROXY_IP:$PORT_P/proxy/api/messages" -H 'Content-Type: application/json' -H 'Content-Type: application/json' -d '{ "sender": "osonoi", "host": "helloworld-proxy:8080" }'

{"id":"3aa4f889-94a0-4be8-b8f2-8b59f8ad3de7","sender":"remko","message":"Hello osonoi (proxy)","host":"helloworld-proxy:8080"}
```


## Apply Network Policy - Allow No Traffic

Define the Network Policy file,

```
$ echo 'apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: helloworld-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress' > helloworld-policy-denyall.yaml
```

Create the Network Policy,

```
$ kubectl create -f helloworld-policy-denyall.yaml
networkpolicy.networking.k8s.io/helloworld-deny-all created
```

Test the `helloworld` and the proxy,
```
$ curl -L -X POST "http://$PUBLIC_IP:$PORT/api/messages" -H 'Content-Type: application/json' -d '{ "sender": "osonoi" }'
curl: (7) Failed to connect to 169.48.67.163 port 31777: Connection timed out

$ curl -L -X POST "http://$PROXY_IP:$PORT_P/proxy/api/messages" -H 'Content-Type: application/json' -H 'Content-Type: application/json' -d '{ "sender": "osonoi", "host": "helloworld-proxy:8080" }'
curl: (7) Failed to connect to 169.48.67.166 port 32333: Connection timed out
```

It takes quite a long time before connections time out. All traffic is denied, despite that we have a LoadBalancer service added to each deployment,

```
$ kubectl get svc
NAME    TYPE    CLUSTER-IP    EXTERNAL-IP    PORT(S)    AGE
helloworld    LoadBalancer    172.21.161.255    169.48.67.163    8080:31777/TCP    3h31m
helloworld-proxy    LoadBalancer    172.21.244.220    169.48.67.166    8080:32333/TCP    11m
kubernetes    ClusterIP    172.21.0.1    <none>    443/TCP    8h
```


## Apply Network Policy - Allow Only Traffic From Proxy

Let's allow traffic to the proxy. 

Define the Network Policy file,

```
$ echo 'apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: helloworld-allow-test
spec:
  selector: app == 'helloworld'
  types:
  - Ingress
  - Engress
  ingress:
  - action: Allow
    protocol: TCP
    destination: 
      ports:
      - 8080' > helloworld-calico-allow-test.yaml
```

Create the Network Policy,

```
$ kubectl create -f helloworld-policy-deny-to-label.yaml
networkpolicy.networking.k8s.io/helloworld-allow-to-label created
```

Test the `helloworld` and the proxy,
```
$ curl -L -X POST "http://$PUBLIC_IP:$PORT/api/messages" -H 'Content-Type: application/json' -d '{ "sender": "osonoi" }'

curl: (7) Failed to connect to 169.48.67.163 port 31777: Connection timed out

$ curl -L -X POST "http://$PROXY_IP:$PORT_P/proxy/api/messages" -H 'Content-Type: application/json' -H 'Content-Type: application/json' -d '{ "sender": "osonoi", "host": "helloworld-proxy:8080" }'

curl: (7) Failed to connect to 169.48.67.166 port 32333: Connection timed out
```


```
kubectl get networkpolicy
kubectl delete networkpolicy helloworld-allow-to-label
```

