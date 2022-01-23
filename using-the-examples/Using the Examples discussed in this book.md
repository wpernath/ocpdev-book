# Using the Examples Discussed in this Book
## Prerequisites
In order to run all of the examples I am discussing in this book, you should have the following software available:

- OpenShift 4.8.x (see below for instructions)
- Maven 3.8.3
- Java JDK 11 or later
- git
- Docker
- The OpenShift client (`oc`) matching the version of the OpenShift Cluster
- An editor to work with (VScode, Eclipse, IntelliJ)

OpenShift needs to have the following Operators installed:
- OpenShift GitOps
- OpenShift Pipelines
- Crunchy Postgres for Kubernetes by Crunchy Data

## Getting an OpenShift instance
This section describes the options you have for using OpenShift.

### Using Red Hat Developer Sandbox
This solution is the easiest one, but unfortunately very limited. You can’t create new projects (namespaces) and you’re not allowed to install additional operators. This solution can be used only for Chapters 1 and 2 and the Helm Chart part of Chapter 3.

To use this option, go to the [Developer Sandbox][1] and register for free.

### Using CodeReady Containers (`crc`)
CodeReady Containers (`crc`) provides a single-node OpenShift installation for Windows, macOS, and Linux. It runs OpenShift on an embedded virtual machine. You have all the flexibility of an external OpenShift cluster without the need for three or more master nodes. You are also able to install additional Operators.

This solution requires the following resources on your local machine:
- 9 GB free memory
- 4 CPU cores
- 50 GB free hard disk space

Go to [GitHub][2] for a list of releases and have a look at the [official documentation][3].


### Using Single Node OpenShift (SNO)
With this solution you have most flexibility in using your OpenShift installation. But this choice also requires most resources. You should have a dedicated spare machine with the following specs in order to use SNO:
- 8 vCPU cores
- 32GB free memory
- 150GB free hard disk space

Visit the [Red Hat Console][4] to start the installation process. After installation, look at my [OpenShift Config script][5], which you can find on GitHub. This script creates persistent volumes, makes the internal registry non-ephemeral, creates a cluster-admin user, and installs necessary operators and a CI environment with Nexus (a maven repository) and Gogs (a Git repository).

## The container image registry

You can use any Docker-compliant registry to store your container images. I am using [Quay.io][6] for all of my images. The account is available for free and doesn’t limit upload/download rates. Once registered, go to **Account Settings—\>User Settings** and generate an encrypted password. Quay.io will give you some options to store your password hash, for example as a Kubernetes-secret, which you can then directly use as push-/pull secrets.

The free account, however, limits you to creating only public repositories. As a result, anybody can read from your repository, but only you are allowed to write and update your image.

Once you’ve created your image in the repository, you have to check the image properties and make sure that the repository is public. By default, Quay.io creates private repositories.

## The structure of the examples
All the examples can be found on GitHub: [https://github.com/wpernath/book-example][7] Please fork the repository and use it as you desire.

### Folders for Chapter 1
The folder `person-service` contains the Java sources of the Quarkus example. If you want to deploy it on OpenShift, please make sure to first install a PostgreSQL server, either via Crunchy Data or by instantiating the `postgresql-persistent` template as follows:

```bash
$ oc new-app postgresql-persistent \
	-p POSTGRESQL_USER=wanja \
	-p POSTGRESQL_PASSWORD=wanja \
	-p POSTGRESQL_DATABASE=wanjadb \
	-p DATABASE_SERVICE_NAME=wanjaserver
```

### Folders for Chapter 2
This chapters introduces Kubernetes configuration and the Kustomize tool. The folders containing the configuration files used in this chapter follow:
- `raw-kubernetes` contains the raw Kubernetes manifest files.
- `ocp-template` contains the OpenShift Template file.
- `kustomize` contains a set of basic files for use with Kustomize.
- `kustomize-ext` contains a set of advanced files for use with Kustomize.

### Folders for Chapter 3
This chapter is about Helm Charts and Kubernetes Operators. The corresponding folders are `helm-chart` and `kube-operator`.

### Folders for Chapter 4
This chapter is about Tekton and OpenShift Pipelines. The sources can be found in the folder `tekton`. Please also have a look at the `pipeline.sh` script. It installs all the necessary Tasks and resources if you call it with the `init` parameter:

```bash
$ pipeline.sh init
configmap/maven-settings configured
persistentvolumeclaim/maven-repo-pvc configured
persistentvolumeclaim/builder-pvc configured
task.tekton.dev/kustomize configured
task.tekton.dev/maven-caching configured
pipeline.tekton.dev/build-and-push-image configured
```

You can start the pipeline by executing:
```bash
$ pipeline.sh start -u wpernath -p <your-quay-token>
pipelinerun.tekton.dev/build-and-push-image-run-20211125-163308 created
```

### Folders for Chapter 5
This chapter is about using Tekton and Argo CD. The sources can be found in the `gitops` folder. To initialize these tools, call:
```bash
$ ./pipeline.sh init [--force] --git-user <user> \
	--git-password <pwd> \
	--registry-user <user> \
	--registry-password <pwd>
```

This call (if given the `--force` flag) creates the following namespaces and Argo CD applications:
- `book-ci`: Pipelines, tasks, and a Nexus instance
- `book-dev`: The current dev stage
- `book-stage`: The last stage release

The following command starts the development pipeline discussed in Chapter 5:
```bash
$ ./pipeline.sh build -u <reg-user> -p <reg-password>
```

Whenever the pipeline is successfully executed, you should see an updated message on the `person-service-config` Git repository. And you should see that Argo CD has initiated a synchronization process, which ends with a redeployment of the Quarkus application.

To start the staging pipeline, call:
```bash
$ ./pipeline.sh stage -r v1.0.1-testing
```

This creates a new branch in Git called `release-v1.0.1-testing`, uses the current dev image, tags it on Quay.io, and updates the `stage` config in Git.

In order to apply the changes, you need to either merge the branch directly or create a pull request and then merge the changes.

[1]:	https://developers.redhat.com/developer-sandbox
[2]:	https://github.com/code-ready/crc/releases
[3]:	https://access.redhat.com/documentation/en-us/red_hat_codeready_containers/1.33/html-single/getting_started_guide/index
[4]:	https://console.redhat.com/openshift/assisted-installer/clusters/~new
[5]:	https://github.com/wpernath/openshift-config
[6]:	https://quay.io/
[7]:	https://github.com/wpernath/book-example
