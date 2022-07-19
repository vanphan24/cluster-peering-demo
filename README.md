# Cluster peering demo

This demo will showcase Cluster Peering which Consul's ability to connect two Consul datacenters together allowing services to be discovered and connected between multiple different Consul datacenters that may be own/managed by multiple different teams.

In our demo, we will deploy Consul datacenters onto two different Kubernetes clusters named **dc1** and **dc2**. We will then deploy a counting app consisting of two different services, **dashboard** and **counting**. Each service will be deployed on different kubernetes cluster. We will create cluster peering connnections between the two Consul datacenters. We will then export the counting service from one Consul datacenter to the other Consul datacenter, allowing the dashboard service to connect to the upstream counting service.

# Pre-reqs

1. You have two Kubernetes clusters available. In this demo example, we will use Azure Kubernetes Service (AKS) but it can be applied to other K8s clusters.
2. Make sure the network between the two kubernetes clusters is routable/connected. For example, if you are deploying each Kubernetes cluster on a different VPC/VNET, make sure you configure VPC/VNET peering between the VPC/VNETs. 
  - If you are using AKS, make sure you select the Azure CNI instead of kubenet.
  - Note these requirements may not not longer be needed by GA.
  
  
  
# Deploy Consul on each Kubernetes cluster.
Note: In our example, we will name our Kubernetes clusters **dc1** and **dc2**.


1. Deploy Consul dc1 to K8s cluster dc1. 
```
kubectl config use-context dc1
```
```
helm install dc1 hashicorp/consul --version "0.45.0" --values consul-values.yaml   
```

2. Deploy Consul dc2 to K8s cluster dc2. 
```
kubectl config use-context dc2
```
```
helm install dc2 hashicorp/consul --version "0.45.0" --values consul-values.yaml   
```

3. Deploy dashboard service on dc1
```
kubectl apply -f countingapp/dashboard.yaml --context dc1
```

4. Deploy counting service on dc2
```
kubectl apply -f countingapp/counting.yaml --context dc2
```

# Create Peering Connections

5. Create Peering Acceptor on dc1 using the provided acceptor-for-dc2.yml file.
```
kubectl apply -f  acceptor-for-dc2.yml --context dc1
```

6. Notice this will create a CRD called peeringacceptors and a secret called peering-token-dc2.
```
kubectl get peeringacceptors --context dc1
```
```
kubectl get secrets --context dc1
```

7. Copy peering-token-dc2 from dc1 to dc2
```
kubectl get secret peering-token-dc2 --context dc1 -o yaml | kubectl apply --context dc2 -f -
```

8. Create Peering Dialer on dc2 using the provided dialer-dc2.yml file.
Note: This step will connect Consul on dc2 to Consul on dc1 using the peering-token
```
kubectl apply -f  dialer-dc2.yml --context dc2
```

9. Export counting services from dc2 to dc1 using the provided exportedsvc-counting.yml file.
This will allow the the counting service to be reachable by the dashboard service in the other Consul datacenter
```
kubectl apply -f countingapp/exportedsvc-counting.yml --context dc2
```

10. Check Consul server on dc1 to see that it can perform health check on counting service on dc2
```
kubectl exec dc1-consul-server-0 --context dc1 -- curl "localhost:8500/v1/health/connect/counting?peer=dc2" | jq
```

Note: If it returns a result, then a peering connection has been established. If there is no returned result, then a misconfig may have occurred. 
- Check that networks between the two Kubernetes clusters are routeable. 
- See Trouble shooting section to check the no errors occurred between dialer and acceptor.
- See Trouble shooting section to 

11. Using your browser, check the dashboard UI and confirm the number displayed is incrementing. Append port **:9002** to the browser URL.
You can get the dashboard UI's public IP address with
```
kubectl get service dashboard --context dc1
```

Example url is **http://20.237.4.200:9002**

If it increments, this means the dashboard is able to reach and display the current count on the counting service.



# Remove Peering Connection

1. Delete the Peering Dialer on dc2.
```
kubectl delete -f  dialer-dc2.yml --context dc2
```
Note: The peering connection is removed but you may want to clean up some of the other resources.

2. Delete peering-token on dc2
```
kubectl delete secret peering-token-dc2 --context dc2
```

3. Delete the Peering Acceptor on dc1. This will also remove the peering-token created on dc1.
```
kubectl apply -f acceptor-for-dc2.yml --context dc1
```

# Optional

You may want to create another Peering Connection to a third Consul deployment on **dc3**. If so, the steps are similar. We will use a different app called **fake service* which has a **frontend** service connecting to a upstream **backend** service.

1. Deploy Consul dc3 to K8s cluster dc2. 
```
kubectl config use-context dc3
```
```
helm install dc3 hashicorp/consul --version "0.45.0" --values consul-values.yaml   
```

2. Deploy frontend service on dc1
```
kubectl apply -f fakeapp/frontend.yaml --context dc1
```

3. Deploy counting service on dc3
```
kubectl apply -f fakeapp/backend.yaml --context dc3
```

# Create Peering Connections

4. Create Peering Acceptor on dc1 using the provided acceptor-for-dc3.yml file.
```
kubectl apply -f  acceptor-for-dc3.yml --context dc1
```

5. Create Peering Dialer on dc2 using the provided dialer-dc2.yml file.
Note: This step will connect Consul on dc2 to Consul on dc1 using the peering-token
```
kubectl apply -f  dialer-dc2.yml --context dc2
```

9. Export counting services from dc3 to dc1 using the provided exportedsvc-backend.yml file.
This will allow the the counting service to be reachable by the dashboard service in the other Consul datacenter
```
kubectl apply -f fakeapp/exportedsvc-backend.yml --context dc3
```

10. Using your browser, check the frontend UI to confirm the frontend service can reach the backend service. Append port **:9090** to the browser URL.
You can get the dashboard UI's public IP address with
```
kubectl get service frontend --context dc1
```
If the box for frontend and backend are in grey, this means the front is able to reach the backend service. If the box are red, then there is no connection. See Troubleshooting section.


# Troubleshooting.

1. Check that there are no errors when trying to establish the peering connection:
```
kubectl logs dc1-consul-server-0 --context dc1 | grep agent.grpc-api.peer
```

```
kubectl logs dc2-consul-server-0 --context dc2 | grep agent.grpc-api.peer
```

2. Check that you have exported your counting service from dc2 to dc1.
```
kubectl logs dc2-consul-server-0 --context dc2 | grep export
2022-07-18T15:02:05.371Z [TRACE] agent.grpc-api.peering.stream.subscriptions: sending public event: peer_id=00345afc-ebf1-a608-e0bd-56cc42e32b23 peer_name=dc1 correlationID=exported-service:counting-sidecar-proxy
2022-07-18T15:02:05.371Z [TRACE] agent.grpc-api.peering.stream.subscriptions: sending public event: peer_id=00345afc-ebf1-a608-e0bd-56cc42e32b23 peer_name=dc1 correlationID=exported-service:counting
```



