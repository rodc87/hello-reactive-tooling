# Hello-World for Platform Tooling

This project is created to test Lightbend's Platform Tooling.

## What's this about?

This project consists of a simple Play application which has [SBT Reactive App](https://github.com/lightbend/sbt-reactive-app) plugin enabled. Using this plugin and the combination of [Reactive CLI](https://github.com/typesafehub/reactive-cli), deployment to a target runtime can be done in a seamless and timely manner.

At the point of writing the target runtime supported is Kubernetes, although the tool might be extended to support DC/OS and other target runtime.

Refer to `DESIGN.md` to see how the application is put together.

## Pre-requisites

* [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) installed and running with insecure registry enabled.
* [Docker](https://docs.docker.com/engine/installation/) command line tools installed and enabled.
* [OpenJDK 8](http://openjdk.java.net/install/) or [Oracle Java 8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* [SBT](http://www.scala-sbt.org/download.html)

### Install Reactive CLI

See the [Platform Tooling](https://s3-us-west-2.amazonaws.com/rp-tooling-temp-docs/deployment-setup.html#install-the-cli) documentation.

## Deploying to Minikube

Setup Minikube Docker environment variables.

```bash
$ eval $(minikube docker-env)
```

Clone this project from Github.

Go into the recently cloned project directory.

```bash
$ cd hello-reactive-tooling
```

Publish the project as Docker images into Minikube.

```bash
$ sbt frontend/docker:publishLocal simple-impl/docker:publishLocal clustered-impl/docker:publishLocal
```

Deploy the `simple-impl` to minikube using the following command. Note for the `simple-impl` we don't generate the Ingress resource as we don't wish to expose the endpoint outside of Kubernetes.

```bash
$ rp generate-kubernetes-deployment hello-reactive-tooling/simple-impl:0.0.1 --generate-services --generate-pod-controllers --env JAVA_OPTS="-Dplay.http.secret.key=simple" | kubectl apply -f -
```

Similarly, deploy the `clustered-impl`. We also won't expose `clustered-impl` outside of Kubernetes. We also scale the `clustered-impl` to `3` instances to test the cluster functionality.

```bash
$ rp generate-kubernetes-deployment hello-reactive-tooling/clustered-impl:0.0.1 --generate-services --generate-pod-controllers --pod-controller-replicas 3 --env JAVA_OPTS="-Dplay.http.secret.key=clustered" --external-service simple-service=_lagom-http-api._tcp.simple-service.default.svc.cluster.local | kubectl apply -f -
```


Deploy the `frontend` to minikube using the following command.

```bash
$ rp generate-kubernetes-deployment hello-reactive-tooling/frontend:0.0.1 --env JAVA_OPTS="-Dplay.http.secret.key=hereiam -Dplay.filters.hosts.allowed.0=$(minikube ip)" | kubectl apply -f -
```



## Accessing the application

Run the following command to access the deployed `frontend`.

```bash
$ curl -vLk "https://$(minikube ip)/"
```

The `frontend` application exposes various other endpoints.

### Accessing the `simple-impl` through `frontend`

Run the following command to access the `simple-impl` through `frontend`.

```bash
$ curl -vLk "https://$(minikube ip)/simple/hello"
```

The `frontend` will invoke the `ServiceLocator` provided by `reactive-lib` to locate `simple-impl` service.

### Accessing the `clustered-impl` through `frontend`

Run the following command to access the `clustered-impl` through `frontend`.

```bash
$ curl -vLk "https://$(minikube ip)/clustered/hello"
```

The `frontend` will invoke the `ServiceLocator` provided by `reactive-lib` to locate `clustered-impl` service.

### Accessing both `clustered-impl` and `simple-impl` through `frontend`

Run the following command.

```bash
$ curl -vLk "https://$(minikube ip)/forward/hello"
```

* The `frontend` will invoke the `ServiceLocator` provided by `reactive-lib` to locate `clustered-impl` service.
* The `clustered-impl` service will invoke Lagom client provided by the `service-api`. Internally Lagom will invoke the `LagomServiceLocator` provided by `reactive-lib`.

### Performing SRV lookup through `frontend`

Run the following command, replacing `<service-name>` with the actual service name _or_ the SRV entry you'd like to find. This command will return a list of address for a given service name.

```bash
$ curl -vLk "https://$(minikube ip)/srv/<service-name>"
```

Example:

```bash
$ curl -vLk "https://$(minikube ip)/srv/_cql._tcp.cassandra-cassandra.cassandra.svc.cluster.local"
```

Run the following command, replacing `<service-name>` with the actual service name and `<endpoint-name>` with the actual endpoint name. This command will return a list of address for a given service name and endpoint name.

```bash
$ curl -vLk "https://$(minikube ip)/srv/<service-name>/<endpoint-name>"
```

Example:

```bash
$ curl -vLk "https://$(minikube ip)/srv/simple-service/lagom-http-api"
```
