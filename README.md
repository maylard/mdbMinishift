[TOC]



# Kubernetes (k8s) MongoDB operator

## background and limitations

- opensource
- others have branded it to orchestration to manage lifecycle containers, yaml, command line, anti affinity rules etc.
- docker containers
- POD group of containers, single service; in our implementation, and instance of mongod.
- A Kubernetes [service](http://kubernetes.io/docs/user-guide/services) serves as an internal load balancer. It identifies a set of replicated [pods](https://docs.okd.io/latest/architecture/core_concepts/pods_and_services.html#pods) in order to proxy the connections it receives to them. Backing pods can be added to or removed from a service arbitrarily while the service remains consistently available, enabling anything that depends on the service to refer to it at a consistent address.
- *Think* of images as cookie cutters and containers as the actual cookies.
- *Think* of OKD as an operating system, images as applications that you run on them, and the containers as the actual running instances of those applications.
- We are using minishift https://github.com/minishift/minishift in the tutorial, allowing us to run OpenShift locally by running a single-node OpenShift cluster inside a VM. You can try out OpenShift or develop with it, day-to-day, on your local host.
- openshift - RedHat ent implementation, Productised version of Kubernetes with web interface. Challenge - built in limitations to single datacenter as Kubernetes original design does not handle high latency, introduces additional complexity and troubleshooting hassle.



# Prerequisites

You will need brew, cask and virtual box at the ready.

### Ops/ Cloud Manager Prerequisites

To install the MongoDB Enterprise Kubernetes Operator, you must:

- Base Url - the url of an Ops Manager instance
- Public API Key - an Ops Manager/ Cloud Manager (CM) Public API Key. Note that you must whitelist the IP range of your Kubernetes cluster so that the Operator may make requests to Ops Manager using this API Key.
  - CM, within the Org, choose Access, API keys, Create API key, and the only permission is Org Owner, make a note of the public key and pvt key, and make sure you have an appropriate API whitelist for this key.

Note: MongoDB Enterprise Kubernetes Operator is compatible with Kubernetes v1.11 or later.

### Base Config files MongoDB Operator

1. Clone the [MongoDB Enterprise Kubernetes Operator repository](https://github.com/mongodb/mongodb-enterprise-kubernetes).

   ```
   git clone https://github.com/mongodb/mongodb-enterprise-kubernetes.git
   ```

2. Note by default, The Kubernetes Operator uses the `mongodb` namespace. To simplify your installation, consider creating a namespace labeled `mongodb` using the following `kubectl` command:



# Setting up minishift

`brew cask install minishift`

`minishift --help`

By default, Minishift uses the driver most relevant to the host OS, it is recommended to specify.

To start:

`minishift start --vm-driver=virtualbox`

OR

`minishift config set vm-driver virtualbox`

```minishift start```

Expect to wait about 10 mins for the creation to complete.

You will be presented with-…

`Creating initial project "myproject" ...Server Information ...OpenShift server started.`

`……`



Use [**minishift oc-env**](https://docs.okd.io/latest/minishift/command-ref/minishift_oc-env.html) to display the command you need to type into your shell in order to add the **oc** binary to your **PATH** environment variable.

`minishift oc-env`

To stop Minishift, use the following command:

```
$ minishift stop
Stopping local OpenShift cluster...
Stopping "minishift"...
```

To terminate environment:  `minishift delete`

Log in as the system admin, cluster admins can create projects and delegate admin rights for the project to any member of the user community. A confusing aspect at this stage is that any username and password combination will be accepted in the web-console. OpenShift Container Platform’s command line interface (CLI) `oc`. Let us stay at the command line for now.

`oc login -u system:admin`

`oc whoami`

A **project** is a Kubernetes namespace with additional annotations, and is the central vehicle by which access to resources for regular users is managed.

create new MongoDB project.

`oc new-project mongodb` 

validate MongoDB project exists.

`oc projects`

Let's add a specific user to the mongodb project. 

`oc adm policy add-role-to-user admin mdb -n mongodb`

`oc describe rolebinding.rbac -n mongodb`

When logged into web interface as mdb, you should now be able to see that Project. Password is cannot be blank, but anything you like. (check reasons for this)

https://<ip>:8443/console/catalog

### Next - need to install the operator.

Now to install MongoDB k8s operator -  https://docs.mongodb.com/kubernetes-operator/stable/tutorial/install-k8s-operator/

`git clone https://github.com/mongodb/mongodb-enterprise-kubernetes.git ` 

Create a pod with the operator running in it...

`oc apply -f crds.yaml`

edit mongodb-enterprise.yaml and add this to the end of the file in line with the `env:`

`- name: MANAGED_SECURITY_CONTEXT `
` value: 'true'`

Also check the securityContext, as this requires a higher value for runAsUser, 1000160000.

```
securityContext:
  runAsNonRoot: true
  runAsUser: 1000160000
```

then:

`oc apply -f mongodb-enterprise.yaml` 

Within your project under applications you should now see the created operator and pod.

# Next create a config map and some secrets:

We will.

1. Have or generate a [Public API Key](https://docs.opsmanager.mongodb.com/current/tutorial/configure-public-api-access/#generate-public-api-key).
2. Grant the user with this new Public API access.
3. Add the IP or CIDR block of any hosts that serve the Kubernetes Operator to the [API Whitelist](https://docs.opsmanager.mongodb.com/current/tutorial/configure-public-api-access/#configure-access-to-whitelisted-ops).

You can enter any of the following:

| Entry                                | Grants                                                 |
| :----------------------------------- | :----------------------------------------------------- |
| An IP address                        | Access to whitelisted operations from that address.    |
| A CIDR-notated range of IP addresses | Access to whitelisted operations from those addresses. |
| `0.0.0.0/0`                          | Unrestricted access to whitelisted operations.         |

If you leave the whitelist empty, you have no access to whitelisted operations.



# Create configmap

edit/ create *configmap-cm.yaml* to the example below with your orgId. Cloud Manager sample *configmap-cm.yaml* file (replace orgId with your cloud manager (CM) org id) which can be obtained from either the URL structure in your browser address bar, or from settings under the CM org page.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-cm
  namespace: mongodb
data:
  projectName: minishift
  orgId: 599eee9xxxxx89f78f769464d3f7d
  baseUrl: https://cloud.mongodb.com
...
```



### apply the configmap

`oc apply -f configmap-cm.yaml`

`oc describe configmaps configmap-cm -n mongodb`

*N.B oc describe command is the name ^ of the config map, not the file name.*

You can check the progress, from OKD, applications, Pods, clicking on your pod name and then Logs tab.

### Create OKD secrets, auth reference

Create a generic secret using the API key from above, edit text between <>:

`oc -n mongodb create secret generic credentials-cm --from-literal="user=<publicKey>" --from-literal="publicApiKey=<apiKey>"`

You can check the creation in OKD under resources, secrets.

# Build replica set definition

Create/ Edit a new yaml file, making sure spec.credentials matches OKD secret reference.

```
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: minishift-rs
  namespace: mongodb
spec:
  members: 3
  version: 4.0.10
  project: configmap-cm
  credentials: credentials-cm
  type: ReplicaSet
  persistent: true
...
```



### create replica set

`oc apply -f replicaset.yaml`

again check progress via pod logs in OKD, as well as the project in cloud manager.

### destroy replica set

`oc delete -f replicaset.yaml`

### connecting mongo

`oc port-forward minishift-rs-0 27017:27017`

^use correct pod name