# Platform meets Openshift

This workshop is focused on basic usage if Kubernetes and troubleshooting for Platform related colleagues. Even though, the main focus will be on troubleshooting it's mandatory to get everyone started.

This workshop is intend to get everyone understand that Platform people already have the necessary level of the low-level techniques going on in Kubernetes. 
This workshop will not make you an Kubernetes expert and I hope that it at least let you change your opinion on getting in touch with Kubernetes.


# Openshift / Kubernetes 
Kubernetes is a Container automation and orchestration platform. Similar to AAP, Kubernetes typically uses YAML file syntax to configure and deploy content. OpenShift Container Platform is Red Hat's Kubernetes distribution.

## Basic configuration

Similar to RHEL's `/etc` Kubernetes uses the `etcd` service to store configuration of all objects configured. Those objects are managed through the `kube-api`using https API calls to CRUD (Create,Read,Update,Delete) all type of configurations. What is specified by Custom Resource Definitions (CRD) providing parameters and options similar to man pages.
`oc` is the Openshift cli extending `kubectl` for some OCP specific use-cases.
`yq` and `jq` are useful to manipulate and compensate returned content into narrowed 
down views.
Follow [this instructions](install-kubernetes.md) for your workshop Kubernetes cluster.

CRD's are comparable to packages in a RHEL system. They provide the definition for functionality which Operators (Ansible playbooks) execute for the Users.
~~~
# debugging
# listing all available CRD's in the cluster
oc get crds 
NAME                        CREATED AT
routes.route.openshift.io   2024-02-01T06:50:11Z
~~~
Reading Linux man pages is the equivalent to reading CRD's. The will provide expected structure, options, values for the User to configure and the Operator to process.
~~~
# debugging
# getting more specific information about the CRD clusterversions
oc get crd routes.route.openshift.io -o yaml | yq -r '.spec | [.group, .names, .scope] '
- route.openshift.io
- kind: Route
  listKind: RouteList
  plural: routes
  singular: route
- Namespaced
~~~
The systemctl `status <name>` is an equivalent of retrieving Custom Resource (CR) objects of what the Operator made out of the CR one added.
~~~
# debugging
# retrieving the object(s)/CR in the API service
oc get service
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.23.0.1    <none>        443/TCP   56m
~~~

## Infrastructure overview

Nodes are typically classified in control-plane/master, infra(infrastructure), worker nodes.
Similar to inventories in Ansible, functionality can mix or separate specific use-cases of workload. 
~~~
# debugging
# list all nodes in the Cluster
oc get nodes
NAME       STATUS   ROLES                         AGE   VERSION
workshop-control-plane   Ready    control-plane   31m   v1.26.0
workshop-worker          Ready    <none>          30m   v1.26.0
workshop-worker2         Ready    <none>          30m   v1.26.0
~~~
labels are used to classify in Kubernetes meaning, we label nodes with their functionality like
* node-role.kubernetes.io/control-plane
* node-role.kubernetes.io/master (deprecated)
* node-role.kubernetes.io/worker
~~~
# debugging
# add functionality to a node
oc label node workshop-control-plane node-role-kubernetes-worker=""

# remove functionality from a node
oc label node workshop-control-plane node-role-kubernetes-worker-
~~~

furthermore, we utilize object.parameters `metadata` for providing information to process and object.parameter `status` on actual object information.
~~~
$ oc get node workshop-worker -o yaml | yq -r .metadata
annotations:
  kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
  node.alpha.kubernetes.io/ttl: "0"
  volumes.kubernetes.io/controller-managed-attach-detach: "true"
creationTimestamp: "2024-02-01T06:48:17Z"
labels:
  beta.kubernetes.io/arch: amd64
  beta.kubernetes.io/os: linux
  kubernetes.io/arch: amd64
  kubernetes.io/hostname: workshop-worker
  kubernetes.io/os: linux
name: workshop-worker
resourceVersion: "5629"
uid: c18e1ef4-8414-4ee4-8303-2fb4802f083c
~~~

the operator updates the object.parameter `status` according to it's functionality (similar to the functionality provided by an Ansible module or any script/binary)
~~~
$ oc get node workshop-worker -o yaml | yq -r '.status | [.allocatable, .nodeInfo]'
- cpu: "8"
  ephemeral-storage: 475461Mi
  hugepages-1Gi: "0"
  hugepages-2Mi: "0"
  memory: 32652836Ki
  pods: "110"
- architecture: amd64
  bootID: ab19026b-73f5-4b93-8b86-80e0ba77b235
  containerRuntimeVersion: containerd://1.6.12
  kernelVersion: 6.2.13-300.mbp.fc38.x86_64
  kubeProxyVersion: v1.26.0
  kubeletVersion: v1.26.0
  machineID: 7041c263dab54cbcb59ffb70fb4f3516
  operatingSystem: linux
  osImage: Ubuntu 22.04.1 LTS
  systemUUID: 7a85f0a6-e032-4fab-98f7-8c237d6d2149
~~~
## from podman container to Kubernetes container
Each and everyone of you is able to deploy a container service on a RHEL system using podman. We typically need following items mandatory to succeed:
* an image name (docker.io/library/alpine:latest, registry.redhat.io/ubi9/ubi:latest)
* a name for the container

Additionally we might want to:
* expose a port to reach a service within the container
* provide ephemeral and or persistent storage
* set environment variables
* provide configuration files 
* set or change user
* set or change security related items (CAPS, SELinux,...)

Kubernetes requires the same mandatory and accepts the same additional items as podman and adds the capability of `namespaces` to have a better separation on similar items deployed multiple times in the same cluster.
The workshop Kubernetes cluster does not require you to login. If you are working on an OCP cluster you need to login first. Kind k8s provides certificate based access through configuration in `~/.kube/config`
~~~
# debugging
# oc login -u username -p password https://api.apps.example.com
~~~

~~~
# debugging
# create your namespace for this workshop
oc create namespace workshop
oc config set-context --current --namespace=workshop
~~~

One main difference between podman and Kubernetes is that Kubernetes expectes a declarative configuration style for easier automation. Meaning, even though we can run ad-hoc pod/containers similar to `ansible -m cmd` we shouldn't do that.

~~~
# debugging
# spin up a nginx web service
podman run -d --name nginx --rm -p 8082:80 docker.io/library/nginx
curl -s -o /dev/null -w '%{http_code}\n' localhost:8082
podman rm -f nginx 
~~~
the same deployment in Kubernetes requires that we configure:
* a deployment (deploymentConfig is deprecated)
* a service (equivalent of podmans port mapping)
* a route (which maps http/https to our service through the Ingress/LoadBalancer)

~~~
# debugging
# spin up a nginx web service pod (no deployment to simplify)
oc -n workshop run nginx --image=docker.io/library/nginx

# we create the service for nginx (port mapping)
cat <<EOF >service.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: workshop
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  sessionAffinity: None
  type: ClusterIP
EOF
oc -n workshop create -f service.yml

# we expose (configure the Router) to accept a FQDN for accessing our service
oc -n workshop expose service nginx --name=nginx --hostname=nginx.apps.example.com --wildcard-policy=None

# if you are using kubectl with an alias to oc you need to inject the yaml instead of using the command
cat <<EOF> route.yml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nginx
  namespace: workshop
spec:
  host: nginx.apps.example.com
  port:
    targetPort: http
  to:
    kind: service
    name: nginx
    weight: null
  wildcardPolicy: None
EOF
oc -n workshop create -f route.yml
~~~
Now we've already touched many items you might consider magic, so I want you to recall what a CRD provides ? That YAML content you didn't know where to get it from.
~~~
# debugging
# if you need descriptions, remove recursive and scroll through with 
# oc explain service.spec.item1
# oc explain service.spec.item1.item2
# oc explain service.spec.item1.item2.item3

# list all options/parameters without description
oc explain service.spec --recursive 
KIND:     Service
VERSION:  v1

RESOURCE: spec <Object>
DESCRIPTION:
FIELDS:
   allocateLoadBalancerNodePorts	<boolean>
   clusterIP	<string>
   clusterIPs	<[]string>
   externalIPs	<[]string>
   externalName	<string>
   externalTrafficPolicy	<string>
   healthCheckNodePort	<integer>
   internalTrafficPolicy	<string>
   ipFamilies	<[]string>
   ipFamilyPolicy	<string>
   loadBalancerClass	<string>
   loadBalancerIP	<string>
   loadBalancerSourceRanges	<[]string>
   ports	<[]Object>
      appProtocol	<string>
      name	<string>
      nodePort	<integer>
      port	<integer>
      protocol	<string>
      targetPort	<string>
   publishNotReadyAddresses	<boolean>
   selector	<map[string]string>
   sessionAffinity	<string>
   sessionAffinityConfig	<Object>
      clientIP	<Object>
         timeoutSeconds	<integer>
   type	<string>
~~~

let's check our nginx deployment now
~~~
# debugging
# check our nginx pod, service and route
oc -n workshop get pod,service,route
NAME        READY   STATUS    RESTARTS   AGE
pod/nginx   1/1     Running   0          10m

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                            PORT(S)   AGE
service/nginx        ClusterIP      172.30.234.184   <none>                                 80/TCP    7m3s

NAME                             HOST/PORT                PATH   SERVICES   PORT   TERMINATION   WILDCARD
route.route.openshift.io/nginx   nginx.apps.example.com          nginx      http                 None
~~~
That looks good but if we try to access the service we will receive a `503` which is the default response for `un` or `mis` configured service access.
~~~
# debugging
curl -s -o/dev/null -w '%{http_code}\n' nginx.apps.example.com
503
~~~
let's troubleshoot why we do not succeed:
* we check if kubernetes was able to pull the image, assign a Node to run and matches security constraints 
    ~~~
    # debugging
    # list events for our namespace and sort them by timestamp 
    oc get events --sort-by='{.metadata.creationTimestamp}'
	LAST SEEN   TYPE     REASON      OBJECT      MESSAGE
	41s         Normal   Scheduled   pod/nginx   Successfully assigned workshop/nginx to workshop-worker
	40s         Normal   Pulling     pod/nginx   Pulling image "docker.io/library/nginx"
	27s         Normal   Pulled      pod/nginx   Successfully pulled image "docker.io/library/nginx" in 13.524697097s (13.524712328s including waiting)
	26s         Normal   Created     pod/nginx   Created container nginx
	26s         Normal   Started     pod/nginx   Started container nginx
    ~~~
* we check the pod, if it's up and running
    ~~~
    # debugging
    oc -n workshop get pod nginx 
    NAME    READY   STATUS    RESTARTS   AGE
    nginx   1/1     Running   0          67s
    ~~~
* we check the service if it matches our pod
    ~~~
    # debugging
    oc -n workshop get service nginx -o yaml | yq -r .spec.selector
    null
    ~~~
    here we already have one difference to podman port mappings, the requirement to match where the port should be mapped to and since we can have multiple containers and or pods to service the port, we need a selector.

Even though there's the capability to `patch` the CR's we use the declarative approach and update our existing service.yml file.
~~~
# debugging
# update the service for nginx (port mapping) with a selector
cat <<EOF >service.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: workshop
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  sessionAffinity: None
  type: ClusterIP
  selector:
    run: nginx
EOF
oc -n workshop replace -f service.yml
~~~
Wait, how do I know the selector and what does it target ? As mentioned before, **metadata**. So even though we did not specify any metadata for our pod, Kubernetes will always add some defaults like
~~~
# debugging
oc -n workshop get pod nginx -o yaml | yq -r .metadata.labels
run: nginx
~~~

Let's re-check the nginx service now 
~~~
# debugging
curl -s -o/dev/null -w '%{http_code}\n' nginx.apps.example.com
200
~~~
Now adding our own labels to our nginx pod we use the declarative approach instead of the ad-hoc one.
~~~
# debugging 
# first, dump the CR which Kubernetes created with our `oc run` command and remove unnecessary or conflicting attributes
oc -n workshop get pod nginx -o yaml  | yq -r 'del(.metadata.annotations,.metadata.creationTimestamp,.metadata.resourceVersion,.metadata.uid,.spec.containers[0].volumeMounts,.spec.imagePullSecrets,.spec.nodeName,.spec.tolerations,.spec.volumes,.status)' > pod.yml

# than delete the nginx pod from the ad-hoc command 
oc -n workshop delete pod nginx 

# update the label section of your pod.yml with a label of your choice
cat pod.yml
apiVersion: v1
kind: Pod
metadata:
  labels:
    nginx: platform-to-ocp # << add a custom label
    run: nginx
  name: nginx
  namespace: default
[.. output omitted ..]

# update the selector in your service.yml matching the custom label added on your pod
cat <<EOF >service.yml
[.. output omitted ..]
  selector:
    nginx: platform-to-ocp
EOF

# now let's apply both CR's at once
# apply get's the object in Kubernetes into the state of desire no matter if they are absent, or existing but different
oc -n workshop apply -f pod.yml -f service.yml

# retry your service and it should still return 200 
curl -s -o/dev/null -w '%{http_code}\n' nginx.apps.example.com
200
~~~

## storage for our workloads
Until now, we did not use any storage explicitly. With the explicit pod deployment for our nginx, we did get the privilege to write the container content in an ephemeral storage (overlayfs). This means, no changes we do are persistent and a rollout or restart will scratch what we did.
Typically ephemeral storage is used for scratch able content like `/var/run`, `/run` or `/tmp`.

The difference between `secrets`,`configMaps` and `persistentStorage` (PV) is that we cannot pre-popluate persistentStorage without modifying the init process of the container.
That is similar to podman's mapping files, directories (secrets/configMaps) or empty volumes (PV).

There are some limitations in the form of size for secrets and configMaps (<2MB) meaning you will not be able to run your database in such configurations and furthermore, those are consider read-only.

For persistentStorage, Kubernetes utilizes StorageClasses which connect to Operators that handle provisioning for us. Those can be similar to RHEL system local paths, Logical Volume's (LVM), Network protocol Services(NFS,CIFS,ISCSI,...) or Storage solutions like Openshift Data Foundation (ODF) providing different features and capabilities.
For easy use, we do not need to understand which provider requires which kind of configuration and instead we create `PersistentVolumeClaims` (PVC) for Storage specifying only the `class`, `size` and expected type `block` or `filesystem`.

~~~
# debugging 
# listing all available StorageClasses in our Cluster
oc get sc
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  36m
~~~
as mentioned before, we can utilize `oc explain` to understand how such a request looks like
~~~
# debugging 
# describe the attributes for mounting persistent Storage
KIND:     PersistentVolumeClaim
VERSION:  v1
RESOURCE: spec <Object>
[.. output omitted ..]

FIELDS:
   accessModes	<[]string>
   dataSource	<Object>
      apiGroup	<string>
      kind	<string>
      name	<string>
   dataSourceRef	<Object>
      apiGroup	<string>
      kind	<string>
      name	<string>
      namespace	<string>
   resources	<Object>
      claims	<[]Object>
         name	<string>
      limits	<map[string]string>
      requests	<map[string]string>
   selector	<Object>
      matchExpressions	<[]Object>
         key	<string>
         operator	<string>
         values	<[]string>
      matchLabels	<map[string]string>
   storageClassName	<string>
   volumeMode	<string>
   volumeName	<string>
~~~

similar as on any RHEL system, sharing storage is something we can but shall not do without having a Cluster Filesystem setup that ensure's, consistency and or fencing. Such request are classified in the PVC through the `accessModes` which are listed below:
* ReadWriteOnce (that should be your default)
* ReadWriteMany (do not use this unless you can accept FS inconsistency)
* ReadOnlyMany (sharing content that doesn't change)
* ReadWriteOncePod (limits the access within the whole Cluster to one Pod)

let's create a PVC for `1GB` in the default StorageClass 
~~~
# debugging 
# create a persistentVolume claim
cat <<EOF> pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
  storageClassName: standard
EOF
oc -n workshop create -f pvc.yml
~~~

now if we check our PV we'll see that nothing happened so far.
~~~
# debugging 
# the PersistentVolume will bind with our Claim once we reference it in a deployment
oc -n workshop get pv,pvc
NAME                          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nginx   Pending                                      standard       39m
~~~

That is due to how the StorageClass defines action once a request is added (WaitForFirstConsumer vs Immediate) which only means, unless we reference the PVC in a Deployment/Pod no action is executed (similar to `podman volume create <name>`) 

Let's reference the PVC for our nginx pod at the default nginx location `/usr/share/nginx/html` 
~~~
# debugging 
# create that reference to our Claim in our nginx pod
cat <<EOF> pod.yml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
    nginx: platform-to-ocp
  name: nginx
  namespace: workshop
spec:
  containers:
    - image: docker.io/library/nginx
      imagePullPolicy: Always
      name: nginx
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
      - name: webdata                     # custom chooseable name
        mountPath: /usr/share/nginx/html  # where to mount the storage
  volumes:
  - name: webdata
    persistentVolumeClaim:
      claimName: nginx                    # the name of our Claim
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
EOF
oc -n workshop delete -f pod.yml
oc -n workshop create -f pod.yml

# podman equivalent command would be
# podman run ... -v volumename:/usr/share/nginx/html -p 8082:80 docker.io/...
~~~

if we now execute our curl command again we'll see an `403` error as there is no index.html content in our documentRoot.
~~~
# debugging 
# try to access an nginx service without content shall return 403 
curl -s -o/dev/null -w '%{http_code}\n' nginx.apps.example.com
403
~~~
Let's create a small "Hello World" index.html file in our persistentVolume.
~~~
# debugging 
# create a custom index page 
oc -n workshop exec -ti nginx -- /bin/bash
root@nginx:/# echo "<h1>Hello Platform</h1>" > /usr/share/nginx/html/index.html
root@nginx:/# exit
~~~
and check our service accordingly
~~~
# debugging 
# check our custom index page
curl -s nginx.apps.example.com
<h1>Hello Platform</h1>
~~~
We could now delete the pod service and route, re-create them and as long as we do not drop our PVC, the content will be there. (dropping the PVC will delete the content as the StorageClass uses `reclaimPolicy: Delete`)

## now into some Storage troubleshooting
We will delete our Pod for now and create a Deployment (similar to an Ansible Role) for creating multiple instances on demand.
~~~
# debugging
# delete the current nginx Pod
oc -n workshop delete -f pod.yml
~~~
A deployment is similar to the Pod definition we already created, but removes hardcoded items like names of the pod (which needs to be unique)
~~~
# debugging
# create a deployment for our nginx pod
cat <<EOF> deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 0
  selector:
    matchLabels:
      app: nginx
      version: v1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  template:                                    # here we start defining our template
    metadata:
      labels:
        app: nginx
        version: v1
    spec:                                      # here we start to configure the same as in our Pod definition
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
              - key: version
                operator: In
                values:
                - v1
            topologyKey: kubernetes.io/hostname
      containers:
      - image: docker.io/library/nginx
        imagePullPolicy: IfNotPresent
        name: nginx
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - name: webdata
          mountPath: /usr/share/nginx/html
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: webdata
        persistentVolumeClaim:
          claimName: nginx
EOF
oc -n workshop create -f deployment.yml
~~~
Now we'll see a slightly different output checking our pod
~~~
# debugging
# check our pod when using deployment
oc -n workshop get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-69bbc98dc6-pmkst   1/1     Running   0          57s
~~~ 
the name used will be tailored with a generated id to scale up and down.
~~~
# debugging
# scale our deployment
oc -n workshop scale --replicas=2 deployment/nginx
~~~
checking on our deployment we will now see that the second Pod is in state `Pending`
~~~
# debugging
# check why scaling doesn't work
oc -n workshop get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE             NOMINATED NODE   READINESS GATES
nginx-694f9bcffd-hmjhb   0/1     Pending   0          86s     <none>        <none>           <none>           <none>
nginx-69bbc98dc6-nnw9m   1/1     Running   0          2m31s   10.245.1.19   node1-worker   <none>           <none>
~~~

Let's evaluate the Kubernetes events on why we cannot scale our second Pod.
~~~
# debugging
# evaluate Kubernetes events for scaling
oc -n workshop get events --sort-by='{.metadata.creationTimestamp}' | tail -5
6m9s        Normal    Started             pod/nginx-69bbc98dc6-nnw9m    Started container nginx
6m9s        Normal    Created             pod/nginx-69bbc98dc6-nnw9m    Created container nginx
5m6s        Warning   FailedScheduling    pod/nginx-694f9bcffd-hmjhb    0/3 nodes are available: 1 node(s) didn't match pod anti-affinity rules, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) had volume node affinity conflict. preemption: 0/3 nodes are available: 1 No preemption victims found for incoming pod, 2 Preemption is not helpful for scheduling..
5m6s        Normal    SuccessfulCreate    replicaset/nginx-694f9bcffd   Created pod: nginx-694f9bcffd-hmjhb
5m6s        Normal    ScalingReplicaSet   deployment/nginx              Scaled up replica set nginx-694f9bcffd to 1
~~~
Kubernetes events are similar to `systemctl status name` output. They will give you a rough idea when Kubernetes itself has a conflict like in our use-case we defined:
* the deployment template specified that there can only be one instance per worker
* the PVC defined that the volume can only be mounted once (where once means worker compared to ReadWriteOncePod)

We could remove the antiAffinity definition (Affinity and antiAffinity are used to avoid SPOF's) or change our PVC to be ReadWriteMany.

In our Lab we will prefer ReadWriteMany as PVC accessMode even though it mean's the already created content will be lost (again `ReclaimPolicy` in the StorageClass)

First, scale down our nginx deployment as known to every System Administrator, you cannot umount a Volume that is in use.
~~~
# debugging
# scale down the deployment to replace the PVC
oc -n workshop scale --replicas=0 deploy/nginx

# delete the PVC and check that PVC and PV will be removed accordingly
oc -n workshop delete -f pvc.yml

# check that our Storage has been deprovisioned
oc -n workshop get pvc,pv
~~~
Now change the PVC to be accessMode `ReadWriteMany`
~~~
cat <<EOF> pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
  storageClassName: standard
EOF
oc -n workshop create -f pvc.yml
~~~
and scale up our nginx deployment again
~~~
oc -n workshop scale --replicas=1 deploy/nginx
~~~
we'll notice that our deployment will not scale as expected and if we investigate the events again we'll see a message explaining the reason
~~~
oc -n workshop get events --sort-by='{.metadata.creationTimestamp}' | tail -5
29s         Normal    SuccessfulCreate       replicaset/nginx-694f9bcffd   Created pod: nginx-694f9bcffd-xdlkp
29s         Normal    ScalingReplicaSet      deployment/nginx              Scaled up replica set nginx-694f9bcffd to 1 from 0
8s          Normal    ExternalProvisioning   persistentvolumeclaim/nginx   waiting for a volume to be created, either by external provisioner "rancher.io/local-path" or manually created by system administrator
14s         Normal    Provisioning           persistentvolumeclaim/nginx   External provisioner is provisioning volume for claim "workshop/nginx"
14s         Warning   ProvisioningFailed     persistentvolumeclaim/nginx   failed to provision volume with StorageClass "standard": Only support ReadWriteOnce access mode
~~~
Depending on the Container Storage Interface (CSI) driver one is utilizing features are available or not. 
Since we are not on a production Cluster, we can utilize a `configMap` instead of a `persistentVolume` 
We need to:
* scale down the deployment
* drop the PV,PVC
* create a configMap containing our index.html
* change the deployment accordingly
~~~
# debugging
# scale down the deployment to be able to delete the PVC
oc -n workshop scale --replicas=0 deploy/nginx

# delete the PVC 
oc -n workshop delete -f pvc.yml

# create a configMap 
oc -n workshop create cm webdata --from-literal=index.html='<h1>Hello Platform</h1>'

# update the deployment volumes to pick the configMap instead of the PVC

cat deployment.yml
[.. output omitted ..]
      volumes:
      - name: webdata
        configMap:              # change from persistentVolumeClaim
          name: webdata         # change from claimName

# and replace the deployment with our new configuration
oc -n workshop replace -f deployment.yml
~~~

now scale the deployment 
~~~
# debugging
# scale the deployment up to 5 instances
oc -n workshop scale --replicas=5 deploy/nginx

# check if we are scaling as expected
oc -n workshop get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-58d65d58b5-5q5h5   1/1     Running   0          4m35s
nginx-58d65d58b5-gws5p   1/1     Running   0          66s
nginx-58d65d58b5-pq49m   0/1     Pending   0          66s
nginx-58d65d58b5-sqf4n   0/1     Pending   0          66s
nginx-58d65d58b5-tb6gl   0/1     Pending   0          66s
~~~

strange, we scaled two Pods but three others are in `Pending` state again.
Evaluating the Kubernetes events once more we'll see the reason for it.
~~~
# debugging
# evaluate the events to understand Pending state of additional Pods
oc -n workshop get events --sort-by='{.metadata.creationTimestamp}' | tail -10
2m53s       Normal    SuccessfulCreate       replicaset/nginx-58d65d58b5   Created pod: nginx-58d65d58b5-sqf4n
2m52s       Warning   FailedScheduling       pod/nginx-58d65d58b5-sqf4n    0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) didn't match pod anti-affinity rules. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod..
2m52s       Normal    Pulled                 pod/nginx-58d65d58b5-gws5p    Container image "docker.io/library/nginx" already present on machine
2m51s       Normal    Started                pod/nginx-58d65d58b5-gws5p    Started container nginx
2m51s       Normal    Created                pod/nginx-58d65d58b5-gws5p    Created container nginx
~~~

The `Warning` message of `FailedScheduling` seems to be hard to digest in the first run but look at it one-by-one:
* 0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }
    this means, we do not allow scheduling of User workload on the control-plane node so out of our three Nodes in the cluster that one cannot be used
* 2 node(s) didn't match pod anti-affinity rules
    remember, we configured our deployment to not run two pods on the same Node for fault tolerance. Two Pods of our scaled deployment have been started using one Node each and the remaining three Pods lack Nodes where there isn't any nginx running already.

The remaining information on the event is the reason of decision making.
So we can either scale our Cluster workers up or remove the AntiAffinity Rule to get a 5 Pod scale working.

to be continued ...
