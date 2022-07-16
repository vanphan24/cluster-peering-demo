# cluster-peering-demo

This demo will showcase Cluster Peering which Consul's ability to connect two Consul datacenters together allowing services to be discovered and routed between multiple different Consul datacenters that may be own/managed by multiple different teams.

In our demo, we will deploy Consul datacenters onto two different Kubernetes clusters. We will then deploy a counting app consisting of two different services, dashboard and counting. Each service will be deployed on different kubernetes cluster. We will create cluster peering connnections between the two Consul datacenters. We will then export the counting service from one Consul datacenter to the other Consul datacenter, allowing the dashboard service to connect to the upstream counting service.

# Pre-reqs

1. You have two Kubernetes clusters available. In this demo example, we will use Azure Kubernetes Service (AKS) but it can be applied to other K8s clusters.
2. Make sure the network between the two kubernetes clusters is routable/connected. For example, if you are deploying each Kubernetes cluster on different VPC/VNETs, make sure you configure VPC/VNET peering between the VPC/VNETS. 
  - On AKS, make sure you select the Azure CNI instead of kubenet.
  - Note these requirements may not not longer be needed by GA.
  
  
  
# Deploy Consul on each Kubernetes cluster.
# Note: 
# In our example, we will name our Kubernetes clusters dc1 and dc2.


1. Deploy Consul dc1 to K8s cluster dc1. 

kubectl config use-context dc1

helm install dc1 hashicorp/consul --version "0.45.0" --values consul-values.yaml   

2. Deploy Consul dc2 to K8s cluster dc2. 

kubectl config use-context dc2

helm install dc2 hashicorp/consul --version "0.45.0" --values consul-values.yaml   

3. Deploy dashboard service on dc1

kubectl apply -f counting/dashboard.yaml --context dc1

4. Deploy counting service on dc2

kubectl apply -f countingapp/counting.yaml --context dc2

# Create Peering Connections

5. Create Peering Acceptor on dc1 using the provided acceptor-for-dc2.yml file.

kubectl apply -f  acceptor-for-dc2.yml --context dc1

6. Notice this will create a CRD called peeringacceptors and a secret called peering-token-dc2.

kubectl get peeringacceptors --context dc1

kubectl get secrets --context dc1

7. Copy peering-token-dc2 from dc1 to dc2

kubectl get secret peering-token-dc2 --context dc1 -o yaml | kubectl apply --context dc2 -f -

8. Create Peering Dialer on dc2 using the provided dialer-dc2.yml file.
Note: This step will connect Consul on dc2 to Consul on dc1 using the peering-token

kubectl apply -f  dialer-dc2.yml --context dc2

9. Export counting services from dc2 to dc1 using the provided exportedsvc-counting.yml file.
This will allow the the counting service to be reachable by the dashboard service in the other Consul datacenter

kubectl apply -f countingapp/exportedsvc-counting.yml --context dc2

9. Check Consul server on dc1 to see that it can perform health check on counting service on dc2

kubectl exec dc1-consul-server-0 --context dc1 -- curl "localhost:8500/v1/health/connect/counting?peer=dc2" | jq

Note: If it returns a result, then a peering connection has been established. If there is no returned result, then a misconfig may have occurred. Check that networks between the two Kubernetes clusters are routeable. 

10. Check the dashboard UI and confirm the number displayed is incrementing. This means the dashboard is able to reach and display the current count on the counting service.

# Remove Peering Connection

1. Delete the Peering Dialer on dc2.

kubectl delete -f  dialer-dc2.yml --context dc2

Note: The peering connection is removed but you may want to clean up some of the other resources.

2. Delete peering-token on dc2

kubectl delete secret peering-token-dc2 --context dc2

3. Delete the Peering Acceptor on dc1. This will also remove the peering-token created on dc1.

kubectl apply -f acceptor-for-dc2.yml --context dc1

# Optional

You may want to create another Peering Connection to a third Consul deployment on dc3. If so, create a Peering Acceptor and new peering token and follow the same process as above.

