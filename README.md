# automata-substrate-devops

## Question 1

For a stateful peer to peer network, we expect high IOPS and high memory usuage. Using an SSD to facilitate faster read/write will help boost performance. However more storage can be attached to the cluster as time goes on in a predictable manner which then leaves us with managing CPU & Memory within the cluster which can be quite unpreditable due to spikes in the network from high computation.

### Vertical Pod Autoscaling

Using a Vertical Pod autoscaling is the best solution because you don't have to run time-consuming benchmarking tasks to determine the correct values for CPU and Memory requests.

Vertical Pod autoscaling ensures the pods in the cluster scales to accommodate network demand. It does this by notifying the cluster autoscaler ahead of an update, and scales the resources (i.e nodes) needed for the resized workload before recreating the pods, this minimizes the disruption time. Given this is a statefulset the network identity (i.e DNS) and persisted state remains intact during the pod recreation.

Horizontal Pod Autoscaling might not be the best scaling method here especially when a fixed set of nodes are required to keep the network running. Also both HPA & VPA dont always work best together as they are competing on the same resources


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
