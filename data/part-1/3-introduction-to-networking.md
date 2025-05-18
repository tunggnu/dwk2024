---
path: '/part-1/3-introduction-to-networking'
title: 'Introduction to Networking'
hidden: false
---

<text-box variant='learningObjectives' name='Learning Objectives'>

After this section you

- are able to use Services to access applications from outside of the cluster

- are able to use Ingresses to access applications from outside of the cluster

</text-box>

Now back to development! Restarting and following logs has been a treat. Next, we'll open an endpoint to the application and access it via HTTP.

#### Simple networking application ####

Let's develop our application so that it has an HTTP server responding with two hashes: a hash that is stored until the process is exited and a hash that is request specific. The response body can be something like "Application abc123. Request 94k9m2". Choose any port to listen to.

I've prepared one [here](https://github.com/kubernetes-hy/material-example/tree/master/app2). By default, it will listen on port 3000.

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app2/manifests/deployment.yaml
  deployment.apps/hashresponse-dep created
```

### Connecting from outside of the cluster ###

We can confirm that the hashresponse-dep is working with the `port-forward` command. Let's see the name of the pod first and then port forward there:

```shell
$ kubectl get po
  NAME                                READY   STATUS    RESTARTS   AGE
  hashgenerator-dep-5cbbf97d5-z2ct9   1/1     Running   0          20h
  hashresponse-dep-57bcc888d7-dj5vk   1/1     Running   0          19h

$ kubectl port-forward hashresponse-dep-57bcc888d7-dj5vk 3003:3000
  Forwarding from 127.0.0.1:3003 -> 3000
  Forwarding from [::1]:3003 -> 3000
```

Now we can view the response from http://localhost:3003 and confirm that it is working as expected.

<exercise name='Exercise 1.05: Project v0.3'>

  Have the project respond something to a GET request sent to the project. A simple html page is good or you can deploy something more complex like a single-page-application.

  See [here](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/) how you can define environment variables for containers.

  Use `kubectl port-forward` to confirm that the project is accessible and works in the cluster by using a browser to access the project.

</exercise>

External connections with Docker used the flag -p `-p 3003:3000` or in Docker compose ports the declaration. Unfortunately, Kubernetes isn't as simple. We're going to use either a *Service* resource or an *Ingress* resource.

#### Before anything else ####

Because we are running our cluster inside Docker with k3d we will have to do some preparations.

Opening a route from outside of the cluster to the pod will not be enough if we have no means of accessing the cluster inside the containers!

```shell
$ docker ps
  CONTAINER ID    IMAGE                      COMMAND                  CREATED             STATUS              PORTS                             NAMES
  b60f6c246ebb    ghcr.io/k3d-io/k3d-proxy:5 "/bin/sh -c nginx-pr…"   2 hours ago         Up 2 hours          80/tcp, 0.0.0.0:58264->6443/tcp   k3d-k3s-default-serverlb
  553041f96fc6    rancher/k3s:latest         "/bin/k3s agent"         2 hours ago         Up 2 hours                                            k3d-k3s-default-agent-1
  aebd23c2ef99    rancher/k3s:latest         "/bin/k3s agent"         2 hours ago         Up 2 hours                                            k3d-k3s-default-agent-0
  a34e49184d37    rancher/k3s:latest         "/bin/k3s server --t…"   2 hours ago         Up 2 hours                                            k3d-k3s-default-server-0
```

After scrolling a bit right we see that K3d has helpfully prepared us the port 6443 to access the API. Also the port 80 is open. All requests to the load balancer here will be proxied to the same ports of all server nodes of the cluster. However, for testing purposes, we'll want _an individual port open for a single node_. Let's delete our old cluster and create a new one with some ports open.

K3d [documentation](https://k3d.io/v5.3.0/usage/commands/k3d_cluster_create/) tells us how the ports are opened, we'll open local 8081 to 80 in k3d-k3s-default-serverlb and local 8082 to 30080 in k3d-k3s-default-agent-0. The 30080 is chosen almost completely randomly, but needs to be a value between 30000-32767 for the next step:

```shell
$ k3d cluster delete
  INFO[0000] Deleting cluster 'k3s-default'
  ...
  INFO[0002] Successfully deleted cluster k3s-default!

$ k3d cluster create --port 8082:30080@agent:0 -p 8081:80@loadbalancer --agents 2
  INFO[0000] Created network 'k3d-k3s-default'
  ...
  INFO[0021] Cluster 'k3s-default' created successfully!
  INFO[0021] You can now use it like this:
  kubectl cluster-info

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app2/manifests/deployment.yaml
  deployment.apps/hashresponse-dep created
```

Above the "agent:0" and "loadbalancer" are based on k3d [documentation](https://github.com/rancher/k3d/blob/main/docs/usage/exposing_services.md) and reading code from [here](https://github.com/rancher/k3d/blob/11cc7979228f304025d61254eb0c0cb2745b9444/cmd/util/filter.go#L119) and [here](https://github.com/rancher/k3d/blob/main/pkg/types/types.go#L65)

Now we have access through port 8081 to our server node (actually all nodes) and 8082 to one of our agent nodes port 30080. They will be used to showcase different methods of communicating with the servers.

<text-box name="Limitations of local environment" variant="hint">

The setup isn't perfect, we will have a limited amount of ports available in the future. This will be sufficient for our use cases.

Your OS may support using the host network so no ports need to be opened. However, I have no experience with this.

</text-box>

#### What is a Service? ####

As *Deployment* resources took care of deployments for us. *Service* resources will take care of serving the application to connections from outside (and also inside!) of the cluster.

Create a file service.yaml into the manifests folder and we need the service to do the following things:

1. Declare that we want a Service
2. Declare which port to listen to
3. Declare the application where the request should be directed to
4. Declare the port where the request should be directed to

This translates into a yaml file with contents

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hashresponse-svc
spec:
  type: NodePort
  selector:
    app: hashresponse # This is the app as declared in the deployment.
  ports:
    - name: http
      nodePort: 30080 # This is the port that is available outside. Value for nodePort can be between 30000-32767
      protocol: TCP
      port: 1234 # This is a port that is available to the cluster, in this case it can be ~ anything
      targetPort: 3000 # This is the target port
```

```shell
$ kubectl apply -f manifests/service.yaml
  service/hashresponse-svc created
```

As we've published 8082 as 30080 we can access it now via http://localhost:8082.

We've now defined a nodeport with `type: NodePort`. [NodePorts](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) are simply ports that are opened by Kubernetes to **all of the nodes** and the service will handle requests in that port. NodePorts are not flexible and require you to assign a different port for every application. As such NodePorts are not used in production but are helpful to know about.

What we'd want to use instead of NodePort would be a *LoadBalancer* type service but this "only" works with cloud providers as it configures a, possibly costly, load balancer for it. We'll get to know them in part 3.

There's one additional resource that will help us with serving the application, *Ingress*.

<exercise name='Exercise 1.06: Project v0.4'>

  Use a NodePort Service to enable access to the project.

</exercise>

#### What is an Ingress? ####

Incoming Network Access resource [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) is a completely different type of resource from *Services*. If you've got your [OSI model](https://en.wikipedia.org/wiki/OSI_model) memorized, it works in layer 7 while services work on layer 4. You could see these used together: first the aforementioned *LoadBalancer* and then Ingress to handle routing. In our case, as we don't have a load balancer available we can use the Ingress as the first stop. If you're familiar with reverse proxies like Nginx, Ingress should seem familiar.

Ingresses are implemented by various different "controllers". This means that ingresses do not automatically work in a cluster, but give you the freedom of choosing which ingress controller works for you the best. K3s has [Traefik](https://containo.us/traefik/) installed already. Other options include Istio and Nginx Ingress Controller, [more here](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

Switching to Ingress will require us to create an Ingress resource. Ingress will route incoming traffic forward to a *Services*, but the old *NodePort* Service won't do.

```shell
$ kubectl delete -f manifests/service.yaml
  service "hashresponse-svc" deleted
```

A [ClusterIP](https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip) type Service resource gives the Service an internal IP that'll be accessible within the cluster.

The following will let TCP traffic from port 2345 to port 3000.

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hashresponse-svc
spec:
  type: ClusterIP
  selector:
    app: hashresponse
  ports:
    - port: 2345
      protocol: TCP
      targetPort: 3000
```

The second resource we need is the new *Ingress*.

1. Declare that it should be an Ingress
2. And route all traffic to our service

**ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dwk-material-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hashresponse-svc
            port:
              number: 2345
```

Then we can apply everything and view the result

```shell
$ kubectl apply -f manifests/
  ingress.networking.k8s.io/dwk-material-ingress created
  service/hashresponse-svc configured

$ kubectl get svc,ing
  NAME                       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
  service/kubernetes         ClusterIP   10.43.0.1    <none>        443/TCP    5m22s
  service/hashresponse-svc   ClusterIP   10.43.0.61   <none>        2345/TCP   45s

  NAME                                             CLASS    HOSTS   ADDRESS                            PORTS   AGE
  ingress.networking.k8s.io/dwk-material-ingress   <none>   *       172.21.0.3,172.21.0.4,172.21.0.5   80      16s
```

We can see that the ingress is listening on port 80. As we already opened port there we can access the application on http://localhost:8081.

<exercise name='Exercise 1.07: External access with Ingress'>

  "Log output" application currently outputs a timestamp and a random string to the logs.

  Add an endpoint to request the current status (timestamp and string) and an ingress so that you can access it with a browser.

  You can just store the string and timestamp to the memory.

</exercise>

<exercise name='Exercise 1.08: Project v0.5'>

  Switch to using Ingress instead of NodePort to access the project. You can delete the ingress of the "Log output" application so they don't interfere with this exercise. We'll look more into paths and routing in the next exercise and at that point you can configure project to run with the "Log output" application side by side.

</exercise>

<exercise name='Exercise 1.09: More services'>

  Develop a second application that simply responds with "pong 0" to a GET request and increases a counter (the 0) so that you can see how many requests have been sent. The counter should be in memory so it may reset at some point.
  Create a new deployment for it and have it share ingress with "Log output" application. Route requests directed '/pingpong' to it.

  In future exercises, this second application will be referred to as "ping-pong application". It will be used with "Log output" application.

  The ping-pong application will need to listen requests on '/pingpong', so you may have to make changes to its code. This can be avoided by configuring the ingress to rewrite the path, but we will leave that as an optional exercise. You can check out https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource

</exercise>
