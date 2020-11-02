## Red Hat DataGrid / Infinispan Cross Site demo on Openshift Container Platform

**  **Updated for On-premise setup** **

This is to document a demo setup for 2 cross site infinispan clusters deployed on 2 Openshift Container Platform. In today's context of multi, hybrid cloud deployments,  I am particularly interested in seeing workloads deployed across multiple 'clouds', especially where states can be preserved; so that we can leverage on the capabilities to explore load distribution or failover usecases.

This was tested on 2 Openshift Container Platform clusters in AWS (using the `LoadBalancer` connector), as well as 2 on premise clusters (using the `NodePort` connector).

This demo uses Red Hat's distribution of the opensource **Infinispan** in-memory cache and **Kubernetes** platform; namely:

- **Red Hat DataGrid** or RHDG, version 8.1 and 
- **Openshift Container Platform** or OCP, version 4.5

The setup is pretty straight forward:
<pre>
  +--------------+                                 +--------------+
  
  | RHDG - c1    |      <--------------------->    | RHDG - c2    |
  
  +--------------+                                 +--------------+
  
+--------------------+                          +--------------------+

|     Openshift      |                          |     Openshift      |

+--------------------+                          +--------------------+
</pre>
c1 : cluster1

c2 : cluster2

### Environment setup

#### On Public Cloud

I have access to 2 running OCP clusters in AWS, all in ap-southeast region. For this demo, having the clusters deployed in a public cloud is **crucial** as I will be using the `LoadBalancer` ingress to connect the 2 RHDG clusters. For on premise setup, the other option is to use a `NodePort`, I have yet to try it (updated in the next section). 
For the LoadBalancer ingress, the Infinispan Operator [checks](https://github.com/infinispan/infinispan-operator/blob/af542c4e456b42a962b1a07fe62f8559c23ec784/pkg/controller/infinispan/xsite.go#L206) for the `Service`'s object `LoadBalancer: ingress` value, so your cloud provider needs to implement it correctly,  hacking it with the `ExternalIPs` of your own load balancer, will not work.


#### On Premise (** updated **)

This was tested using 2 OCP clusters that are installed on premise. After the initial test which failed (Got a `Generic error in managing x-site coordinators` error message). I tore down everything and setup from scratch. This time round, I let the clusters negotiate the x-site handshakes a little longer and I found that the clusters did connect to each other.

    $ oc logs -f example-infinispan-0 | grep x-site
    15:36:50,046 INFO  (jgroups-5,example-infinispan-0-11602) [org.infinispan.XSITE] ISPN000439: Received new x-site view: [c1]
    15:37:01,112 INFO  (jgroups-5,example-infinispan-0-11602) [org.infinispan.XSITE] ISPN000439: Received new x-site view: [c1, c2]



#### Setting up the base environment

- Install RHDG

RHDG installation is done via the Operator Lifecycle Manager on OCP, official docs [here](https://access.redhat.com/documentation/en-us/red_hat_data_grid/8.1/html/running_data_grid_on_openshift/index)

I will not be detailing the steps here, as it is pretty straightforward.

After you installed the operator, proceed with the following steps:


- Create identical namespaces / projects on both clusters

        oc new-project rhdg-cluster

- on cluster1, create service account `c1`

        oc create sa c1

- Grant role to service account, the document says grant `view` access for the project but I encountered a lot of permission issues during the initial setup, not wanting to lose the big picture of setting up the cross site deployment, I decided to use a full admin role, since this is just a demo :( 

        oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:rhdg-cluster:c1

- You may want to extract the token and save it, for later use
  
        oc sa get-token c1 > c1.txt

- Same goes for cluster2, c2, create namespace, create service account, grant role and extract the token out 

        oc new-project rhdg-cluster

        oc create sa c2

        oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:rhdg-cluster:c2

        oc sa get-token c2 > c2.txt

- then on BOTH clusters, generate the secrets, this is for the current project to access the other cluster

        oc create secret generic c1-token --from-literal=token=$(cat c1.txt)
    
        oc create secret generic c2-token --from-literal=token=$(cat c2.txt)


OK, the scaffolding stuffs should be done by now. Next step is to deploy the RHDG cluster on both sides, as we have already deployed the operator. 

#### Installing the Infinispan clusters using Operators

We will now construct the CRD for the 2 clusters, which is pretty much symmetrical. Do note that for cross site to work, both namespaces and cluster name across the OCP clusters needs to be the same.

The CRD yaml definitions are as follows:

#### A) On public cloud (AWS)

- c1: replace the url endpoints before using

```
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: example-infinispan
spec:
  replicas: 2
  service:
    type: DataGrid
    sites:
      local:
        name: c1
        expose:
          type: LoadBalancer
      locations:
        - name: c1
          url: openshift://<cluster1 kubernetes api endpoint>:<port>  # e.g. openshift://api.mycluster.example.com:6443 
          secretName: c1-token
        - name: c2
          url: openshift://<cluster1 kubernetes api endpoint>:<port>
          secretName: c2-token
  logging: 
    categories:
      org.infinispan: trace
      org.jgroups: trace
      org.jgroups.protocols.TCP: debug
      org.jgroups.protocols.relay.RELAY2: debug      
  expose: 
    type: Route
  
```
- c2: replace the url endpoints before using

```
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: example-infinispan
spec:
  replicas: 2
  service:
    type: DataGrid
    sites:
      local:
        name: c2
        expose:
          type: LoadBalancer
      locations:
        - name: c1
          url: openshift://<cluster1 kubernetes api endpoint>:<port>  # e.g. openshift://api.mycluster.example.com:6443 
          secretName: c1-token
        - name: c2
          url: openshift://<cluster1 kubernetes api endpoint>:<port>
          secretName: c2-token
  logging: 
    categories:
      org.infinispan: trace
      org.jgroups: trace
      org.jgroups.protocols.TCP: debug
      org.jgroups.protocols.relay.RELAY2: debug      
  expose: 
    type: Route
```

#### B)** Updated **For On Premise, the CRD definitions I used are appended

- c1: replace the url endpoints before using, note the sites are exposed as NodePorts. According to documents, the NodePort connector only works on `the same network`, I believe it means in the same subnet, which is true in my demo setup.

```
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: example-infinispan
spec:
  replicas: 2
  service:
    type: DataGrid
    sites:
      local:
        name: c1
        expose:
          type: NodePort
          nodePort: 32556
      locations:
        - name: c1
          url: openshift://api.apps.ocpcluster1.<yourdomain>.com:6443
          secretName: c1-token
        - name: c2
          url: openshift://api.apps.ocpcluster2.<yourdomain>.com:6443
          secretName: c2-token
  logging: 
    categories:
      org.infinispan: trace
      org.jgroups: trace
      org.jgroups.protocols.TCP: debug
      org.jgroups.protocols.relay.RELAY2: debug      
  expose: 
    type: Route
```

- c2: replace the url endpoints before using, note the sites are exposed as NodePorts.

```
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: example-infinispan
spec:
  replicas: 2
  service:
    type: DataGrid
    sites:
      local:
        name: c2
        expose:
          type: NodePort
          nodePort: 32616
      locations:
        - name: c2
          url: openshift://api.apps.ocpcluster2.<yourdomain>.com:6443
          secretName: c2-token
        - name: c1
          url: openshift://api.apps.ocpcluster1.<yourdomain>.com:6443
          secretName: c1-token
  logging: 
    categories:
      org.infinispan: trace
      org.jgroups: trace
      org.jgroups.protocols.TCP: debug
      org.jgroups.protocols.relay.RELAY2: debug      
  expose: 
    type: Route
```


### Deploying the cluster

- save the yaml contents above (either the cloud or on prem ones) into a yaml file and deploy them to the OCP clusters (c1, c2) respectively


e.g. on cluster 1

        oc create -f c1-cluster.yaml 
    
Upon successful deployment of the RHDG cluster, you should be able to see one operator pod, 2 RHDG pods

        NAME                                  READY   STATUS    RESTARTS   AGE
        example-infinispan-0                  1/1     Running   0          39m
        example-infinispan-1                  1/1     Running   0          38m
        infinispan-operator-668d7c565-s49jb   1/1     Running   0          10h
    
The following services should be generated as well

        NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP                                       PORT(S)          AGE
        example-infinispan        ClusterIP      172.30.27.101    <none>                                            11222/TCP        10h
        example-infinispan-ping   ClusterIP      None             <none>                                            8888/TCP         10h
        example-infinispan-site   LoadBalancer   172.30.142.140   <elb endpoint>.ap-southeast-1.elb.amazonaws.com   7900:30256/TCP   10h

As I chose a `Route` endpoint for the cluster, you will be able to see a route object

        NAME                          HOST/PORT          PATH   SERVICES             PORT    TERMINATION   WILDCARD
        example-infinispan-external   <route hostname>          example-infinispan   <all>   passthrough   None

To ensure the cross-site setup is successful, check the logs on both clusters, they should look similar to this (from cluster 2):

        $ oc logs -f example-infinispan-0 | grep x-site
        15:34:04,172 INFO  (jgroups-100,example-infinispan-0-64507) [org.infinispan.XSITE] ISPN000439: Received new x-site view: [c2]
        15:34:04,378 INFO  (jgroups-98,example-infinispan-0-64507) [org.infinispan.XSITE] ISPN000439: Received new x-site view: [c1, c2]

You can also see a `configmap` generated to hold the configuration of the infinispan cluster


#### Creating the caches

The 2 cross site clusters are deployed, but we have yet to create an actual cache to hold data. We will look at that now.

There are a few ways to create the cache, 

- using the Operator, which exposes a Infinispan Cache api for you to do that.
- `oc rsh` into the pod and use the cli tool to create the caches.
- I went the easy way , doing it via the web console.

The web / admin console of the RHDG cluster can be access via the exposed route. i.e. `oc get route` to see it.

Use the hostname (port 80) of the route to access the admin console. You will be prompted for the credentials and if you have yet to get them, go to the terminal and run this:  

        $ oc get secret example-infinispan-generated-secret -o jsonpath="{.data.identities\.yaml}" | base64 --decode
        credentials:
        - username: developer
          password: xxxxxxxx
        - username: operator
          password: xxxxxxxx

Use the `operator` credentials to login

Using the web console, `Data Container / Create Cache` 

![create cache](images/dg-3.png "Create Cache")

![create cache](images/dg-4.png "Create Cache")


- Paste this following xml config to create a cache in cluster 1

```
    <infinispan>
      <cache-container>
        <distributed-cache name="cache-xsite">
          <encoding media-type="application/x-protostream"/>
          <backups>
            <backup site="c2" strategy="SYNC">
              <take-offline min-wait="120000"/>
            </backup>
          </backups>
        </distributed-cache>
      </cache-container>
    </infinispan>
```
- Likewise for cluster 2

```
    <infinispan>
      <cache-container>
        <replicated-cache name="cache-xsite">
          <encoding media-type="application/x-protostream"/>
          <backups>
            <backup site="c1" strategy="ASYNC" >
                <take-offline min-wait="120000"/>
              </backup>
          </backups>
        </replicated-cache>
      </cache-container>
    </infinispan>
```

#### Test Drive

You can use a Hotrod springboot client I have written [here](https://github.com/wohshon/application-clients/tree/master/rhdg-springboot) to test the app. The demo app defines a 'Person' object and uses protobuf to send the value to RHDG.The springboot app wraps around the CRUD usecases with a rest api call.

- I point the app to cluster 1, put in a few values and is able to retrieve it via cluster 2
- you can use the web console to verify the cross cluster synchronization as well.

Other things to be aware of, as the cache uses the protostream media type, to use the web console to view the cache entries, you will need to define the Protobuf Schema (e.g. the demo.PersonEntity protobuf scheme in the sample code) before the web console actually works.

![create protobuf schema](images/dg-5.png "Create ProtoBuf Schema")

![Seach entries](images/dg-6.png "Search")


##### Failover

For single site failover, as long as I keep one running RHDG pod, no data will be lost. 

If I kill off all replicas on one side, the behaviour I observed is that all data will be lost when the POD came back up. Not sure if there a configuration to automate the data sync, something I will have to find out.

To sync the data from backup site, I need to go over to the backup site's web console, initiate a Transfer to push the data over to the recovered cluster.

We have just scratched the surface of a simple cross site setup, hope this is useful! That's all for now!

-- Woh Shon 24 OCT 2020










