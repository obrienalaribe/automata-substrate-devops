# automata-substrate-devops

## Question 1

For a stateful peer to peer application such as this, there would be a lot of replication required in production which means high IOPS is of more importance than CPU. Using an SSD to facilitate faster read/write will be essential. Multithreading or having multiple CPU cores might not be as crucial here. Memory will need to be high as well

Vertical Pod autoscaling in Auto mode
Due to Kubernetes limitations, the only way to modify the resource requests of a running Pod is to recreate the Pod. If you create a VerticalPodAutoscaler object with an updateMode of Auto, the VerticalPodAutoscaler evicts a Pod if it needs to change the Pod's resource requests. Given that this application is a statefulset order and network identity (i.e dns) is maintained during this scale up event which in turn shouldnt affect the p2p network

To limit the amount of Pod restarts, use a Pod disruption budget. To ensure that your cluster can handle the new sizes of your workloads, use cluster autoscaler and node auto-provisioning.

Vertical Pod autoscaling notifies the cluster autoscaler ahead of the update, and provides the resources needed for the resized workload before recreating the the workload, to minimize the disruption time.

Using a Vertical Pod autoscaling is the best solution because you don't have to run time-consuming benchmarking tasks to determine the correct values for CPU and memory requests.

This means Reduced maintenance time because the autoscaler can adjust CPU and memory requests over time on both the cluster and the node without any manual intervention

## Question 2

#### Deployment Frequency
- CI best practices and considerations
- Trunk Based Development & Feature Toggles for faster SDLC and time to market
- Utilize events where possible to automate build process
- Use Tags not branches on build artifacts (Git SHA or SemVer)
- Validate & Test code before building a final image, this means validation/testing should be baked in the build process i.e Dockerfile
- CD best practices and considerations
- Secure your deployment key i.e use short-lived keyless auth
- Manage your k8s YAML in version control for easier transparency and collaboration across team
- Utilize Gitops where possible & limit access to server
- Monitor state of new release via release gates to determine if a rollback is required

#### Lead Time for Changes
- The ability to work on new ideas independently/autonomous without having to get permission from outside of the team
- Leaving the details to those doing the work
- Shifting security & testing left
- Utilizing a cross-functional team working in parallel

#### Time to Restore Services
- Infrastructure as Code to make environments are reproducible and consistent in case for HA & DR
- Incident Management process for managing disruptions and restoring services within agreed SLAs.
- Introducing SLIs, SLOs, SLAs as well as Error Budgets
- Understanding the RTO & RPO of the business
- Ability to rollback to a previous working state i.e ensuring persisted data is restorable from a remote source

#### Change Failure Rate
- Practice chaos engineering and restore targeted services
- Using observability tools to track Traces, Logs & Metrics
- Monitor Golden Signals (Uptime, Latency, Error Rate, Throughput)
 

## Question 3

The network issue is caused by a blocked firewall rule. Running the command below shows a set of firewall rules on the container

Run ```iptables -L``` inside the container
 
```
[root@15a4ab069ec1 ~]# iptables -L 
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
 
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
 
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DROP       tcp  --  anywhere             anywhere             tcp dpt:http
 
```

#### Root Cause Analysis

The above firewall rule is set to DROP only HTTP request but seems to work for HTTPS. Testing ICMP also proved that the domain is reachable which further confirms that this is not a DNS problem

#### Solution

Understanding the RCA, there's 2 potential solutions here:

##### 1. **Disable Firewall rule:** 

```
iptables -D OUTPUT -p tcp -m tcp --dport 80 -j DROP
```

Running the above command will disable the firewall rule and temporarily resolve the issue. However given images are immutable if the container was restarted the problem will still exist. So this solution doesnt scale well

##### 2. **Use a Forward Proxy:** 

This can be achieved in the following stages:
a. Run a squid container inside the same default docker network with command below
```
docker run -itd --name proxy ubuntu/squid 
``` 
b. Rerun the faulty container again with HTTP_PROXY set using the env flag
```
docker run -it --env http_proxy="http://172.17.0.2:3128" --env https_proxy="http://172.17.0.2:3128"  --privileged atactr/devops-assignment:1.0.0
```
c. Retest the curl command inside the faulty container 
```
curl -v www.google.com
```

Running through the steps above tunnels the request to www.google.com via the proxy container. This is a better solution since it makes each container configurable which can scale across multiple containers. This proxy value can also be configured in `~/.docker/config.json` or via the Docker Daemon
