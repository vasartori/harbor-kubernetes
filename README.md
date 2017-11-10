# VMWare Harbor "hard way"

All components of harbor was separeted in directories (with a intuitive name :p);
The files have sufix, it represents:
 - dpl = Deployment
 - svc = Service
 - pvc = Persistent Volume Claim
 - secret = Secrets (passwords, certificates...)

# What you NEED do to deploy

The manifests are configured in a way that it should deploy and work as is on minikube.

If you're not using minikube, LOOK inside of all files and change what makes sense for your environment like:
 - URL's
 - Certificates (you must generate all)

## URLs

The deployment is set up to use https://harbor.192.168.99.100.xip.io. This should
resolve to your minikube deployment (`$ minikube ip` to confirm). If your machine
is offline you may need to add it to `/etc/hosts`. If you are not using minikube
you can update this URL as needed in the included Kubernetes manifests.


## Certificates

Self signed certificates have been created for the domain `harbor.192.168.99.100.xip.io` and have been added to the various secret files. If you need to change the domain you can generate new secrets using `$ docker run -ti -e SSL_SUBJECT="harbor.192.168.99.100.xip.io" -e SSL_IP=192.168.99.100 -e OUTPUT=k8s paulczar/omgwtfssl` (replace the subject and IP as needed) and copy output into `ingress/secret.yaml`.

## Ingress and Dynamic Volume Claims

Ideally your Kubernetes cluster should support both ingress and dynamic volumes.

If your cluster does not support Dynamic Volume Claims you may need to
modify the storage manifests for each service that requires it (`mysql/mysql-pvc.yaml` and `registry/registry-pvc.yaml`). You may also need to uncomment and set the storage class annotations in them.

If your cluster does not support ingress you can try to use `kubectl create -f nginx` instead of
`kubectl create -f ingress` when following the install instructions.

If you are using minikube then you'll want to ensure that `ingress` and `default-storageclass` plugins are enabled. If not you should enable them. Sometimes for minikube's ingress to work correctly you need to stop and start minikube again.

```
$ minikube addons enable ingress
$ minikube addons enable default-storageclass
```

# Example Deployment to Minikube

Install the latest versions of

* [minikube](https://github.com/kubernetes/minikube/releases)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-via-curl)

Start minikube and ensure its set up correctly for ingress and storageclass:

```
$ minikube start                                     
Starting local Kubernetes v1.8.0 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.

$ minikube addons list
- default-storageclass: enabled
- ingress: enabled
```

Deploy Harbor to Kubernetes:

```
$ for i in adminserver ingress jobservice mysql registry ui; do kubectl create -f $i; done
configmap "adminserver-config" created
deployment "adminserver" created
...
...
service "ui" created
```

After a few minutes confirm everything looks good:

```
$ kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
adminserver-7d95b54c9-mhtkg   1/1       Running   0          2m
jobservice-545b888ffd-p8g54   1/1       Running   1          2m
mysql-85786d8c5f-6jsrf        1/1       Running   0          2m
registry-9c48dfbd7-jrdzv      1/1       Running   0          2m
ui-9d68b87cf-wmkrg            1/1       Running   2          2m
```

Then try to access the UI via your browser - [https://harbor.192.168.99.100.xip.io](https://harbor.192.168.99.100.xip.io). Accept the self signed cert.

Next try to login to the docker registry ( user `admin` password `Harbor12345`):

```
$ docker login harbor.192.168.99.100.xip.io
Username (admin): test
Password:
Error response from daemon: Get https://harbor.192.168.99.100.xip.io/v1/users/: x509: certificate signed by unknown authority
```

It should fail. By default docker expects registries to be signed by a trusted Certificate Authority. The certificate used by our ingress service is self signed. You can tell docker to trust our self signed certificate by doing:

```
$ sudo mkdir /etc/docker/certs.d/harbor.192.168.99.100.xip.io
$ sudo cp ingress/ca.crt /etc/docker/certs.d/harbor.192.168.99.100.xip.io/

$ docker login harbor.192.168.99.100.xip.io
Username (admin): admin
Password:
Login Succeeded
$ docker pull alpine
Using default tag: latest
latest: Pulling from library/alpine
Digest: sha256:d6bfc3baf615dc9618209a8d607ba2a8103d9c8a405b3bd8741d88b4bef36478
Status: Image is up to date for alpine:latest
$ docker tag alpine harbor.192.168.99.100.xip.io/library/alpine
$ docker push harbor.192.168.99.100.xip.io/library/alpine
The push refers to a repository [harbor.192.168.99.100.xip.io/library/alpine]
2aebd096e0e2: Pushed
latest: digest: sha256:ca559e07aab1445b3aaf1c7d1996a6b0fcc0df3557461ae582459680effd33dc size: 528
```
