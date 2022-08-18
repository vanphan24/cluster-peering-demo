# Cluster peering demo

This demo will showcase Cluster Peering which Consul's ability to connect two Consul datacenters together allowing services to be discovered and connected between multiple different Consul datacenters that may be own/managed by multiple different teams. To create a peering connection, we must establish one datacenter as the *Acceptor* and one as the *Dialer*. The Accepter will create a token, which will be given to the Dialer. The Dialer will use the token to connect to the Acceptor, which then establish a Peering connection.

In our demo, we will deploy Consul datacenters onto two different Kubernetes clusters named **dc1** and **dc2**. We will then deploy a counting app consisting of two different services, **dashboard** and **counting**. Each service will be deployed on different kubernetes cluster. We will create cluster peering connnections between the two Consul datacenters. We will then export the counting service from one Consul datacenter to the other Consul datacenter, allowing the dashboard service to connect to the upstream counting service.

![alt text](https://github.com/vanphan24/cluster-peering-demo/blob/main/images/Screen%20Shot%202022-08-18%20at%2010.40.40%20AM.png "Cluster Peering Demo")

# Pre-reqs

1. You have two Kubernetes clusters available. In this demo example, we will use Azure Kubernetes Service (AKS) but it can be applied to other K8s clusters.

    Note: 
    - If using AKS, you can use the Kubenet CNI or the Azure CNI. The Consul control plane and data plane will use Load Balancers to communicate between Consul datacenters.
    - Since Load Balancers are used on both control plane and data plane, each datacenter can reside on different networks (VNETS, VPCs). No direct network connections (ie peering connections) are required. 
3. Add or update your hashicorp helm repo:

```
helm repo add hashicorp https://helm.releases.hashicorp.com
```
or
```
helm repo update hashicorp
```
  
  
  
# Deploy Consul on each Kubernetes cluster.
Note: In our example, we will name our Kubernetes clusters **dc1** and **dc2**.


1. Deploy Consul dc1 to K8s cluster dc1. 
```
kubectl config use-context dc1
```
```
helm install dc1 hashicorp/consul --version "0.47.1" --values consul-values.yaml                                  
```

2. Deploy Consul dc2 to K8s cluster dc2. 
```
kubectl config use-context dc2
```
```
helm install dc2 hashicorp/consul --version "0.47.1" --values consul-values.yaml                                  
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

5. Create Peering Acceptor on dc1 using the provided acceptor-for-dc2.yaml file.  
Note: This step will establish dc1 as the Acceptor.
```
kubectl apply -f  acceptor-on-dc1-for-dc2.yaml --context dc1
```

6. Notice this will create a CRD called *peeringacceptors* and a secret called *peering-token-dc2*.
```
kubectl get peeringacceptors --context dc1
NAME   SYNCED   LAST SYNCED   AGE
dc2    True     2m46s         2m47s
```
```
kubectl get secrets --context dc1
```

7. Copy *peering-token-dc2* from dc1 to dc2.
```
kubectl get secret peering-token-dc2 --context dc1 -o yaml | kubectl apply --context dc2 -f -
```

8. Create Peering Dialer on dc2 using the provided dialer-dc2.yaml file.  
Note: This step will establish dc2 as the Dialer and will connect Consul on dc2 to Consul on dc1 using the peering-token.
```
kubectl apply -f  dialer-dc2.yaml --context dc2
```

9. Confirm on the Consul web UI that a peering connection has been established:

    a. Run *kubectl get service --context dc1*  to retreive the Consul's UI's EXTERNAL-IP address.  
    b. Open a browser and go to the Consul UI using the Consul UI's EXTERNAL-IP address.   
    c. Navigate to the ***Peers*** tab on the left hand side and confirm that ***dc2*** is shown as a peer.   


    ![alt text](https://github.com/vanphan24/cluster-peering-demo/blob/main/images/Screen%20Shot%202022-08-18%20at%2011.28.22%20AM.png)



10. Export counting services from dc2 to dc1 using the provided exportedsvc-counting.yaml file.  
Note: On the *data plane*, this will allow the the counting service to be reachable by the dashboard service in the other Consul datacenter
```
kubectl apply -f countingapp/exportedsvc-counting.yaml --context dc2
```

11.  Confirm on the Consul web UI on ***dc1*** that the *counting* service is now visible on the ***Services*** tab. Notice that it shows the counting service is imported from ***dc2***

![alt text](https://github.com/vanphan24/cluster-peering-demo/blob/main/images/Screen%20Shot%202022-08-18%20at%201.15.40%20PM.png)


Alternatively, you can run the below command, which will check that Consul server on dc1 can perform health check on counting service on dc2
```
kubectl exec dc1-consul-server-0 --context dc1 -- curl "localhost:8500/v1/health/connect/counting?peer=dc2" | jq
```

Note: If it returns a result, then a peering connection has been established. If there is no returned result, then a misconfig may have occurred. 

- See Trouble shooting section to check the no errors occurred between dialer and acceptor.




12. Using your browser, check the dashboard UI and confirm the number displayed is incrementing. Append port **:9002** to the browser URL.
You can get the dashboard UI's EXTERNAL IP address with
```
kubectl get service dashboard --context dc1
```

Example url is **http://20.237.4.200:9002**

If it increments, this means the dashboard is able to reach and display the current count on the counting service. This confirms that the peering connection is also established on the *data plane*.

![alt text](https://github.com/vanphan24/cluster-peering-demo/blob/main/images/Screen%20Shot%202022-07-18%20at%209.08.49%20PM.png)


# Remove Peering Connection

1. Delete the Peering Dialer on dc2.
```
kubectl delete -f  dialer-dc2.yaml --context dc2
```
Note: The peering connection is removed but you may want to clean up some of the other resources.

2. Delete peering-token on dc2
```
kubectl delete secret peering-token-dc2 --context dc2
```

3. Delete the Peering Acceptor on dc1. This will also remove the peering-token created on dc1.
```
kubectl delete -f acceptor-on-dc1-for-dc2.yaml --context dc1
```

# Optional - Connect to another Consul datacenter.

You may want to create another Peering Connection to a third Consul deployment on **dc3**. If so, the steps are similar. We will use a different app called **fake app** which has a **frontend** service connecting to a upstream **backend** service.

![alt text](https://github.com/vanphan24/cluster-peering-demo/blob/main/images/Screen%20Shot%202022-07-18%20at%208.59.15%20PM.png)


1. Deploy Consul dc3 to K8s cluster dc2. 
```
kubectl config use-context dc3
```
```
helm install dc3 hashicorp/consul --version "0.47.1" --values consul-values.yaml   
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

4. Create Peering Acceptor on dc1 using the provided acceptor-on-dc1-for-dc3.yaml file.
```
kubectl apply -f  acceptor-on-dc1-for-dc3.yaml --context dc1
```
5. Copy peering-token-dc3 from dc1 to dc3
```
kubectl get secret peering-token-dc3 --context dc1 -o yaml | kubectl apply --context dc3 -f -
```


5. Create Peering Dialer on dc3 using the provided dialer-dc3.yaml file.
Note: This step will connect Consul on dc2 to Consul on dc1 using the peering-token
```
kubectl apply -f  dialer-dc3.yaml --context dc3
```

9. Export counting services from dc3 to dc1 using the provided exportedsvc-backend.yaml file.
This will allow the the counting service to be reachable by the dashboard service in the other Consul datacenter
```
kubectl apply -f fakeapp/exportedsvc-backend.yaml --context dc3
```

10. Using your browser, check the frontend UI to confirm the frontend service can reach the backend service. Append **:9090/ui** to the EXTERNAL-IP address for the browser URL.
You can get the dashboard UI's EXTERNAL IP address with
```
kubectl get service frontend --context dc1
```
If frontend and backend boxes are in white, this means the front is able to reach the backend service. If the boxes are red, then there is no connection. See Troubleshooting section.

![alt text](https://github.com/vanphan24/cluster-peering-demo/blob/main/images/Screen%20Shot%202022-07-18%20at%209.11.37%20PM.png)

# Troubleshooting.

1. Check that there are no errors when trying to establish the peering connection:
```
kubectl logs dc1-consul-server-0 --context dc1 | grep agent.grpc-api.peer
```

```
kubectl logs dc2-consul-server-0 --context dc2 | grep agent.grpc-api.peer
```

```
kubectl logs dc3-consul-server-0 --context dc3 | grep agent.grpc-api.peer
```

If no errors in the logs, the peering connection is likelt established.

2. Check that you have exported your counting service from dc2 to dc1.
```
kubectl logs dc2-consul-server-0 --context dc2 | grep export
2022-07-18T15:02:05.371Z [TRACE] agent.grpc-api.peering.stream.subscriptions: sending public event: peer_id=00345afc-ebf1-a608-e0bd-56cc42e32b23 peer_name=dc1 correlationID=exported-service:counting-sidecar-proxy
2022-07-18T15:02:05.371Z [TRACE] agent.grpc-api.peering.stream.subscriptions: sending public event: peer_id=00345afc-ebf1-a608-e0bd-56cc42e32b23 peer_name=dc1 correlationID=exported-service:counting
```

3. Check that dc1 can reach perform health check on upstream service of dc2 or dc3
```
kubectl exec dc1-consul-server-0 --context dc1 -- curl "localhost:8500/v1/health/connect/counting?peer=dc2" | jq
```
```
kubectl exec dc1-consul-server-0 --context dc1 -- curl "localhost:8500/v1/health/connect/backend?peer=dc3" | jq
```
Example output:
```
kubectl exec dc1-consul-server-0 --context dc1 -- curl "localhost:8500/v1/health/connect/counting?peer=dc2" | jq
. . .
  {
    "Node": {
      "ID": "50c562c8-8031-b57d-8d26-a4b8c99526aa",
      "Node": "aks-agentpool-34660438-vmss000000",
      "Address": "10.3.1.133",
      "Datacenter": "dc1",
      "PeerName": "dc2",
. . .
```

4. If checks in previous steps look ok but your dashboard or frontend service cannot reach the counting or backend services, respectively, check that your dashboard or frontend service has the correct annotations on deployment, and the upstream URL is set to *localhost:<port>*

Example for the counting app: 
The upstream annotation is pointing to *'counting.svc.dc2.peer:9001'*
```
      annotations:
        'consul.hashicorp.com/connect-inject': 'true'
        'consul.hashicorp.com/transparent-proxy': 'false'
        'consul.hashicorp.com/connect-service-upstreams': 'counting.svc.dc2.peer:9001'
```

The upstream URL is set to *localhost:9001*  
```
  spec:
      containers:
      - name: dashboard
        image: hashicorp/dashboard-service:0.0.4
        ports:
        - containerPort: 9002
        env:
        - name: COUNTING_SERVICE_URL
          value: 'http://localhost:9001'
```
  
