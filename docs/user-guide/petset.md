---
---

* TOC
{:toc}

__Terminology__

Throughout this doc you will see a few terms that are sometimes used interchangeably elsewhere, that might cause confusion. This section attempts to clarify them.

* Node: A single virtual or physical machine in a Kubernetes cluster.
* Cluster: A group of nodes. Unless mentioned otherwise a cluster is a single failure domain (i.e one availability zone/cloud provider).
* Persistent Volume Claim: A request for storage, typically a [persistent volume](/docs/user-guide/persistent-volumes/walkthrough.md).
* Host name: The hostname attached to the UTS namespace of the pod, i.e the output of running `hostname` in the pod.
* DNS/Domain name: A *cluster local* domain name resolvable by any pod in the cluster using standard DNS tools like `nslookup` and `dig`.
* Ordinality: the proprety of being "ordinal", or occupying a position in a sequence.
* Pet: a single member of a PetSet; more generally, a stateful application. The use of this term is predated by the well understood "Pets vs Cattle" analogy.

## What is PetSet?

In Kubernetes, most pod management abstractions group them into disposable units of work that compose a micro service. Replication controllers for example, are designed with a weak guarantee - that there should be N replicas of a particular pod template. The pods are treated as stateless units, if one of them is unhealthy or superseded by a better version, the system just disposes it. But there's a class of clustered applications that don't adhere to this model, and managing them is notoriously hard because:
* They require stronger notions of identity and membership which they use in internal protocols like leader election and replication.
* They are especially prone to race conditions and deadlocks during startup, scaling and update.

PetSet is designed to help with this problem by modifying the original "stateless replicas" model in a few important ways. It bestows upon each replica, or "Pet" the following properties:
* a stable hostname, available in DNS
* an ordinal index
* stable storage: linked to the ordinal & hostname
* discovery of peers for quorum
* startup/teardown ordering

The rest of this document describes how these properties are used in deploying clustered applications.

## Alpha limitations

Before you start deploying applications with PetSet, there are a few limitations you should understand.
* PetSet is an *alpha* resource, not available in any Kubernetes release prior to 1.3.
* As with all alpha/beta resources, it can be disable through the `--runtime-config` option passed to the apiserver, and in fact most likely will be disabled on hosted offerings of Kubernetes.
* Updating an existing PetSet is currently a manual process, meaning you either need to deploy a new PetSet with the new image version, or orphan Pets one by one, update their image, and join them back to the cluster.
* The only updatable fields on a PetSet is the `replicas` field.
* The storage for a given pet is provisioned by a dynamic storage provisioner based on the requested `storage class`. Note that dynamic volume provisioning is also currently in alpha.
* Deleting the PetSet  *will not*  delete all pets. You will either have to manually scale it down to 0 pets first, or delete the pets yourself.
* Deleting and/or scaling a PetSet down will *not* delete the volumes associated with the PetSet. This is done to ensure safety first, your data is more valuable than an auto purge of all related PetSet resources. Deleting the Persistent Volume Claims will result in a deletion of the associated volumes.
* All PetSets currently require a "governing service", or a Service responsible for the network identity of the pets. The user is responsible for this Service.

## Deploying applications as PetSets

A minimal PetSet might look like:

```yaml
# A headless service to create DNS records
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  # *.nginx.default.svc.cluster.local
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1alpha1
kind: PetSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
    spec:
      terminationGracePeriodSeconds: 0
      containers:
      - name: nginx
        image: gcr.io/google_containers/nginx-slim:0.7
        ports:
        - containerPort: 80
          name: web
        command:
        - nginx
        args:
        - -g
        - "daemon off;"
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
      annotations:
        volume.alpha.kubernetes.io/storage-class: anything
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi

```

For completeness it includes the `Service` definition. An astute observer might notice that it looks strikingly similar to the manifest of an RC or DaemonSet because it uses the same constructs: labels, selectors and a podTemplate. The easiest way to explain how you can use the features of a PetSet is by walking through some basic examples.

### Identity preservation

Lets create the PetSet shown in the previous section just to see what it does. This toy example illustrates how PetSets preseve the identity of a pod:

```shell
$ kubectl create -f petset.yaml
service "nginx" created
petset "nginx" deleted
```

Notice that you got a bunch of different resources.
2 pods with predictable names, because the PetSet has `replicas` set to 2. The format of the names is `$(petset name)-$(ordinal index assigned by petset controller)`:

```shell
$ kubectl get po
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          10m
web-1     1/1       Running   0          10m
```

A headless Service that provides the network identity for each pod, this is the Service included in the yaml:

```shell
$ kubectl get svc
NAME          CLUSTER-IP     EXTERNAL-IP       PORT(S)   AGE
nginx         None           <none>            80/TCP    12m
```

2 persistent volumes, one per pod. This is auto created by the PetSet based on the `volumeTemplate` field in the yaml:

```shell
$ kubectl get pv
NAME                                       CAPACITY   ACCESSMODES   STATUS    CLAIM               REASON    AGE
pvc-90234946-3717-11e6-a46e-42010af00002   1Gi        RWO           Bound     default/www-web-0             11m
pvc-902733c2-3717-11e6-a46e-42010af00002   1Gi        RWO           Bound     default/www-web-1             11m
```

This actually covers 3/5 PetSet properties: stable hostname, ordinal index and stable storage. To see how, try the following test.
The containers are running nginx webservers, which by default will look for an index.html file in `/usr/share/nginx/html/index.html`. That directory is backed by a PersistentVolume created by the PetSet. So lets write our hostname there (remember the PetSet gives us a stable hostname):

```shell
$ for i in 0 1; do kubectl exec web-$i -- sh -c 'echo $(hostname) > /usr/share/nginx/html/index.html'; done
```

delete all pods in the petset:
```shell
$ kubectl delete po -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```

Wait for them to come back up, and try to retrieve the previously written hostname through the DNS name of the peer (remember the PetSet also gives us stable storage, and that the hostname is linked to the DNS name).
```shell
$ kubectl exec -it web-1 -- curl web-0.nginx
web-0
$ kubectl exec -it web-0 -- curl web-1.nginx
web-1
```

Lets dissect the manifest. First the governing Service:
```yaml
# A headless service to create DNS records
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  # *.nginx.default.svc.cluster.local
  clusterIP: None
  selector:
    app: nginx
```

* It's a headless Service that selects over all pets
* It controls a domain defined by `$(service name).$(namespace).svc.cluster.local`, where "cluster.local" is the [cluster domain].
* Every pet gets the DNS name: `$(petname).$(governing service suffix)`, where the governing service is defined by the `serviceName` field on the PetSet.

So in the example above we had a pet with the DNS name: `web-0.nginx.default.svc.cluster.local`.

The next interesting piece is the `annotations` field of the PetSet.

```yaml
annotations:
  pod.alpha.kubernetes.io/initialized: "true"
```

This field is more of a debugging hook. It pauses any scale up/down operations on the entire PetSet. If you'd like to pause a petset after each pet, set it to `false` in the template, wait for each pet to come up, verify it has initialized correctly, and then set it to `true` using `kubectl edit` on the pet (setting it to `false` on *any pet* is enough to pause the PetSet). If you don't need it, create the PetSet with it set to `true` as shown.

Everything else in the PetSet spec is identical to a ReplicationController, except for the `volumeClaimTemplates` section. Each member of this list is tantamount to saying: give each pet some storage defined by this template.

```yaml
  volumeClaimTemplates:
  - metadata:
      name: www
      annotations:
        volume.alpha.kubernetes.io/storage-class: anything
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

* The name on this template, `www`,  matches the name of a `volumeMount` on the PetSet.
* The template has a `storage-class` annotation, this is simply to get the attention of the [dynamic pv provisioner]. If you don't specify this annotation, you will have to create volumes for you Pets by hand.

### Runtime environment initialization

To explain how you can initialize the runtime environment of a container, we will simulate virtual machines with PetSets. VMs are hard to simulate with containers because they rely on package managers like `apt-get` and `yum` for installation, which assume a certain layout of the filesystem embedded in the dpkg or rpm. When you create a docker image, you are essentially packing the root filesystem into the base image. We'd like to use an existing rootfs, but carry over any packages installed by the user at runtime through a shared volume in a way that the user hardly notices they're running in a container. PetSet already gives us a consistent identity as show above, so all we need is a way to initialize the user environment before allowing tools like `kubectl exec` to enter the application container. This brings us to [init containers], a resource that helps pods initialize their runtime environment.

{% include code.html language="yaml" file="petset_vm.yaml" ghlink="/docs/user-guide/petset_vm.yaml" %}

You can create this PetSet:

```shell
$ kubectl create -f ./petset_vm.yaml
service "ub" created
petset "vm" created
```

This should give you 2 pods.
```shell
$ kubectl get po
NAME      READY     STATUS     RESTARTS   AGE
vm-0      1/1       Running    0          37s
vm-1      1/1       Running    0          2m
```

We can exec into it and install nginx
```shell
$ kubectl exec vm-0 /bin/sh
# apt-get update
...
# apt-get install nginx -y
```

On killing this pod we need it to come back with all the PetSet properties, as well as the installed nginx packages.
```shell
$ kubectl delete po vm-0
pod "vm-0" deleted

$ kubectl get po
NAME      READY     STATUS    RESTARTS   AGE
vm-0      1/1       Running   0          1m
vm-1      1/1       Running   0          4m
```

Now you can exec back into vm-0 and start nginx
```shell
$ kubectl exec -it vm-0 /bin/sh
# mkdir -p /var/log/nginx /var/lib/nginx; nginx -g 'daemon off;'

```
And access it from anywhere in the cluster (and because this is an example that simulates vms, we're going to apt-get install netcat too)
```shell
$ kubectl exec -it vm-1 /bin/sh
# printf "GET / HTTP/1.0\r\n\r\n" | netcat vm-0.ub 80
```

Now to analyze the yaml, the only different section is the init container itself:
```yaml
        pod.alpha.kubernetes.io/init-containers: '[
            {
                "name": "rootfs",
                "image": "ubuntu:15.10",
                "command": [
                    "/bin/sh",
                    "-c",
                    "for d in usr lib etc; do cp -vnpr /$d/* /${d}mnt; done;"
                ],
                "volumeMounts": [
                    {
                        "name": "usr",
                        "mountPath": "/usrmnt"
                    },
                    {
                        "name": "lib",
                        "mountPath": "/libmnt"
                    },
                    {
                        "name": "etc",
                        "mountPath": "/etcmnt"
                    }
                ]
            }
        ]'

```

* Init containers are an alpha feature part of a GA resource, so it's encoded as a json-string blob in the annotations
* Init containers run sequentially and *before* the app container
* In this example, we can't copy files over in an entrypoint script, because we want to copy them into volumes that are mounted into "sensitive" locations

### Finding peers

In both the examples till now we just used kubectl to get the names of existing pods, and as humans, we could tell which ones belonged to a given PetSet. Ideally Pets in a PetSet need to be able to find their peers. One way to do so is by contacting the API server, just like `kubectl`, but that has several disadvantages:
* It makes any startup script non-portable, users need to write specialized code for Kubernetes
* It introduces several points of failure, the secrets to talk to the API server might not be mounted, the Service fronting the apiserver might be in flux etc
* It introduces a upgrade related failure scenario, since you're deserializing raw resources from the apiserver
* It puts load on the apiserver

PetSet gives you a way to disover your peers using DNS records. To illustrate this we can use the previous example.

```shell
$ kubectl exec -it vm-0 /bin/sh
# apt-get install dnsutils
...
# nslookup -type=srv ub.default
Server:		10.0.0.10
Address:	10.0.0.10#53

ub.default.svc.cluster.local	service = 10 50 0 vm-1.ub.default.svc.cluster.local.
ub.default.svc.cluster.local	service = 10 50 0 vm-0.ub.default.svc.cluster.local.
```

There is a [peer finder]() program that we will use in later examples to achieve a similar result.

TODO elaborate on how to use the peer finder

### Master slave

A common application deployment patters is a master/slave setup. This involves statically choosing a single shard as the master, and parenting every other shard to the master, as a slave. The instructions for most master/slave setups begin with: contact a admin, and end with: run to the hills. Jokes aside, this is because deploying master/slave is a little delicate. The steps are:
* Start 1 database as the master
* Find it's ip
* Start all other databases pointed at the master ip

We're intentionally ignoring the failover scenario.
If you think about what the admin is doing:

1. Making sure the first database is started in stand alone mode, meaning it doesn't need any input about peers, and no peers are trying to join it
2. Waiting till it's healthy
3. Checking its ip
4. Starting one more database with the master ip
5. goto step 2

A meticulous admin would re-verify that the master ip hasn't changed on each iteration, and that all previous shards continue to believe they are slaves of the current master. The PetSet controller tries to mimic these actions. It will always start one pet at a time and wait for it to enter Running and Ready before starting the next one. Instead of using ips, it provisions DNS records for each pet, so even if the IP does change the DNS name will not.

The [redis example](https://github.com/kubernetes/kubernetes/tree/master/test/e2e/testing-manifests/petset/redis) implements this pattern.

### Quorum

Quorum based systems are more dynamic than simple master/slave deployment. They can usually tolerate n-1/2 members going down, because the remaining members form a quorum and vote on accepting writes.

The [zookeeper example](https://github.com/kubernetes/kubernetes/tree/master/test/e2e/testing-manifests/petset/zookeeper) implements this pattern.

TODO fill in this section

### Active active

The [mysql galera example](https://github.com/kubernetes/kubernetes/tree/master/test/e2e/testing-manifests/petset/mysql-galera) implements this pattern.

TODO fill in this section


### Decentralized quorum

TODO We don't have an example yet for this, but it's worth describing.

## Future Work

## Alternatives


