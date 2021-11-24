# Chapter Three: Packaging with Helm and Kubernetes Operators
In this chapter, you'll learn more about working with container images. In particular, I will help you understand the concepts of Helm charts and Kubernetes Operators.

## Modern distributions
As already mentioned in Chapter 2, the most difficult task when turning code into a useful product is creating a distributable package from the application. It is no longer just a matter of zipping all files together and putting them somewhere. We have to take care of several meta-artifacts. Some of these might not belong to the developer, but to the owner of the application.

Distribution takes place on two levels:
- *Internal* distribution: making a containerized application available to the IT team within your own organization. Even if this task requires "only" fitting into the company's CI/CD chain, this step can take some training. It is the subject of Chapter 4.
- *External* distribution: making your containerized application available for customers or other third parties. That is the subject of this chapter.

These types of distribution have much in common. In fact, before you can make my apps available for others (external distribution), you might have to put it into the local CI/CD chain (internal distribution). Kubernetes is there to automate most of the tasks.
 
## Using an external registry with Docker or Podman
As already briefly described, in order to externally distribute an application in the modern, generally expected manner, you need a repository. The repository could be either a public image repository such as [Quay.io](https://quay.io/ "Red Hat Quay") or [Docker Hub](https://hub.docker.com "Docker Hub"), or a private repository that is accessible by your customers. This chapter uses the [Quay.io](https://quay.io/ "Red Hat Quay") public repository for its examples, but the principles and most of the commands apply to other types of repositories as well.

You can easily get a free account on Quay.io, but it must be publicly accessible. That means everybody can read your repositories, but only you or people whom you specifically grant permissions to can write to the repositories.

### The Docker and Podman build tools
Along with creating an account on Quay.io, install either [Docker](https://www.docker.com "Docker") or [Podman](https://podman.io/getting-started/installation "Podman") on your local machine. This section offers an introduction to building and image and uploading it to your repository with those tools

#### Docker versus Podman and Buildah
Docker was a game-changing technology when it was first introduced in the early 2010 decade. It made containers a mainstream technology, before Google released Kubernetes.

However, Docker requires a daemon to run on the system hosting the tool, and therefore must be run with root (privileged) access. This is usually unfeasible in Kubernetes, and certainly on Red Hat OpenShift.

Podman was then invented and became a popular replacement for Docker. Podman performs basically the same tasks as Docker and has a compatible interface where almost all the commands and arguments are the same. Podman is very lightweight, and — crucially — can be run as an ordinary user account without a daemon.

However, Podman currently runs only on GNU/Linux. If you are working on a Windows or macOS system, you have to [set up a remote Linux system to use Podman](https://github.com/containers/podman/blob/master/docs/tutorials/mac_win_client.md "Setting up remote podman").

Podman internally uses [Buildah](https://buildah.io/) to build container images. According to the official [GitHub page](https://github.com/containers/buildah "Buildah"), Buildah is the main tool for building images that conform to the [Open Container Initiative (OCI)](https://opencontainers.org/ "Open Container Initiative") standard. The documentation states, "Buildah's commands replicate all the commands that are found in a Dockerfile. This allows building images with and without a Dockerfile, without requiring any root privileges."

To build a container image inside OpenShift (for example, using [Source-to-Image (S2I)](https://github.com/openshift/source-to-image "Source-to-Image") or a Tekton pipeline), you should directly use Buildah instead.

Because everything you need to do with Buildah is part of Podman anyway, there is no CLI client for macOS or Windows available. The documentation states to use Podman instead.

#### Your first build
With Docker or Podman in place, along with a repository on Quay.io or another service, check out the [demo repository for this article](https://github.com/wpernath/book-example "Demo App Git Repo"). In `person-service/src/main/docker` you can find the file that configures builds for your application. The file is called simply `Dockerfile`. To build the demo application, based on the Quarkus Java framework, change into the top-level directory containing the demo files. Then enter the following commands, plugging in your Quay.io username and password:

```bash
$ docker login quay.io -u <username> -p <password>
$ mvn clean package -DskipTests
$ docker build -f src/main/docker/Dockerfile.jvm -t quay.io/wpernath/person-service .
```

The first command logs into your Quay.io account. The second command builds the application with [Maven](https://maven.apache.org/what-is-maven.html "Maven"). The third, finally, creates a docker image out of the app. If you choose to use Podman, simply substitute `podman` for `docker` in your Docker commands, because Podman supports the same arguments. You could also make scripts that invoke Docker run with Podman instead, by aliasing the string `docker` to `podman`:

```bash
$ alias docker=podman
```

Setting up Podman on any non-Linux system is a little bit tricky, as mentioned. You need access to a Linux system, running either directly on one of your systems or in a virtual machine. The Linux system basically works as the execution unit for the Podman client. The [documentation mentioned earlier](https://github.com/containers/podman/blob/master/docs/tutorials/mac_win_client.md "Setting up remote podman") shows you how that works.

When your image is ready, you need to push the image to your repository:

```bash
$ docker push quay.io/wpernath/person-service -a
```

This will push all (`-a`) locally stored images to the repository, including all tags. Now you can use the image in your OpenShift environment (Figure 1).

![Image 1: The person-service on quay.io after you've pushed it via docker push ](Bildschirmfoto%202021-10-26%20um%2008.19.12.png)
And that's it. This workflow works for all Docker-compliant registries.

Because Docker or Podman are only incidental to this chapter, I will move on to our main topics and let you turn to the many good articles out on the Internet to learn about how to build images.

#### Testing the image
Now that you have your image stored in Quay.io, test it to see whether everything has successfully worked out. For this, you are going to use our Kustomize example from the previous chapter.

First of all, make sure that the `kustomize-ext/overlays/dev/kustomization.yaml` file looks like this:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base

namePrefix: dev-
commonLabels:
  variant: development


# replace the image tag of the container with latest
images:
- name: image-registry.openshift-image-registry.svc:5000/book-dev/person-service:latest
  newName: quay.io/wpernath/person-service
  newTag: latest

# generate a configmap
configMapGenerator:
  - name: app-config
    literals:
      - APP_GREETING=We are in DEVELOPMENT mode

# this patch needs to be done, because kustomize does not change
# the route target service name
patches:
- patch: |-
    - op: replace
      path: /spec/to/name
      value: dev-person-service
  target:
    kind: Route

```

Then simply execute the following commands to install your application:

```bash
$ oc login <your openshift cluster>
$ oc new-project book-test
$ oc apply -k kustomize-ext/overlays/dev
configmap/dev-app-config-t9m475fk56 created
service/dev-person-service created
deployment.apps/dev-person-service created
route.route.openshift.io/dev-person-service created
```

The resulting event log should look like Image 2.
![Image 2: Event log for dev-person-service ](Bildschirmfoto%202021-10-26%20um%2009.57.18.png)

If you're on any other operating system than Linux, Podman is a little bit complicated to use right now. The difficulty is not with about the command-line (CLI) tool. The `podman` CLI in most cases is identical to the `docker` CLI. Instead, the difficulty lies in installation, configuration, and integration into non-Linux operating systems.

This is unfortunate, because Podman is much more lightweight than Docker. And Podman does not require root access. So if you have some time—or are already developing your applications on Linux—try to set up Podman. If not, continue to use Docker.

#### Working with Skopeo
[Skopeo](https://github.com/containers/skopeo "Skopeo") is another command line tool that helps you work with container images and different image registries without the heavyweight Docker daemon. On macOS, you can easily install Skopeo via `brew`:

```bash
$ brew install skopeo
```

You can use `skopeo` to tag an image in a remote registry:

```bash
$ skopeo copy \
   docker://quay.io/wpernath/person-service:latest \
   docker://quay.io/wpernath/person-service:v1.0.1-test
```

But you can also use Skopeo to mirror a complete repository by copying all images in one go.

### Next steps after building the application
Now that your image is stored in a remotely accessible repository, you can start thinking about how to let a third party easily install your application. Two mechanisms are popular for this task: the [Helm chart](https://helm.sh "Helm chart") project and a [Kubernetes Operator](https://operatorframework.io "Kubernetes Operator"). The rest of this chapter introduces these tools.

## Helm Charts
Think about Helm charts as a package manager for Kubernetes applications, like RPM or Deb for Linux. Once a Helm chart is created and hosted on a repository, everyone is able to install, update, and delete your chart from a running Kubernetes installation.

Let's first have a look on how to create a Helm chart to install your application on OpenShift.

First of all, you need to download and [install the Helm CLI](https://helm.sh/docs/intro/install/ "Install Helm"). On macOS you can easily do this via:

```bash
$ brew install helm
```

Helm has its own directory structure for storing necessary files. Helm allows you to create a basic template structure with everything you need (and even more). The following command creates a Helm chart structure for a new chart called `foo`:

```bash
$ helm create foo
```

But I think it's better not to create a chart from a template, but to start from scratch. To do so, enter the following commands to create a basic file system structure for your new Helm chart:

```bash
$ mkdir helm-chart
$ mkdir helm-chart/templates
$ touch helm-chart/Chart.yaml
$ touch helm-chart/values.yaml
```

Done. This creates the directory structure for your chart. You now have to put some basic data into `Chart.yaml`, using the following example as a guide to plugging in your own values:

```yaml
apiVersion: v2
name: person-service
description: A helm chart for the basic person-service used as example in the book
home: https://github.com/wpernath/book-example
type: application
version: 0.0.1
appVersion: "1.0.0"
sources:
    - https://github.com/wpernath/book-example
maintainers:
    - name: Wanja Pernath
      email: wpernath@redhat.com
```

You are now done with your first Helm chart. Of course, right now it does nothing special. You have to fill the chart with some content. So now copy the following files from the previous chapter into the `helm-chart/templates` folder:

```bash
$ cp kustomize-ext/base/*.yaml helm-chart/templates/
```

The directory structure of your Helm chart now looks like this:
```bash
$ tree helm-chart
helm-chart
├── Chart.yaml
├── templates
│   ├── config-map.yaml
│   ├── deployment.yaml
│   ├── route.yaml
│   └── service.yaml
└── values.yaml

1 directory, 6 files
```

### Packaging, installing, upgrading and rolling back your Helm chart
Now that you have a very simple Helm chart, you can package it:

```bash
$ helm package helm-chart
Successfully packaged chart and saved it to: person-service-0.0.1.tgz
```

And the following command installs the Helm chart into a newly created OpenShift project called `book-helm1`:

```bash
$ oc new-project book-helm1
$ helm install person-service person-service-0.0.1.tgz
NAME: person-service
LAST DEPLOYED: Mon Oct 25 17:10:49 2021
NAMESPACE: book-helm1
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

If you go now to the OpenShift console, you should see the Helm release (Image 3).
![Image 3: OpenShift Console with our Helm Chart](Bildschirmfoto%202021-10-25%20um%2017.12.05.png)

You can get the same overview via the command line. The first command that follows shows a list of all installed Helm charts in your namespace. The second command provides the update history of a given Helm chart.

```bash
$ helm list
$ helm history person-service
```

If you create a newer version of the chart, upgrade the release in your Kubernetes namespace by entering:
```bash
$ helm upgrade person-service person-service-0.0.5.tgz
Release "person-service" has been upgraded. Happy Helming!
NAME: person-service
LAST DEPLOYED: Tue Oct 26 19:15:07 2021
NAMESPACE: book-helm1
STATUS: deployed
REVISION: 7
TEST SUITE: None
```

The `rollback` argument helps you to return to a specified revision of the installed chart. First get the history of the installed chart by entering:
```bash
$ helm history person-service
REVISION	UPDATED                 	STATUS    	CHART               	APP VERSION	DESCRIPTION
1       	Tue Oct 26 19:24:10 2021	superseded	person-service-0.0.5	v1.0.0-test	Install complete
2       	Tue Oct 26 19:24:46 2021	deployed  	person-service-0.0.4	v1.0.0-test	Upgrade complete
```

Then roll back to revision 1 by entering:
```bash
$ helm rollback person-service 1
Rollback was a success! Happy Helming!
```

The history shows you that the rollback was successful:
```bash
$ helm history person-service
REVISION	UPDATED                 	STATUS    	CHART               	APP VERSION	DESCRIPTION
1       	Tue Oct 26 19:24:10 2021	superseded	person-service-0.0.5	v1.0.0-test	Install complete
2       	Tue Oct 26 19:24:46 2021	superseded	person-service-0.0.4	v1.0.0-test	Upgrade complete
3       	Tue Oct 26 19:25:48 2021	deployed  	person-service-0.0.5	v1.0.0-test	Rollback to 1
```

If you are not satisfied with a given chart, you can easily uninstall it by calling:
```bash
$ helm uninstall person-service
release "person-service" uninstalled
```

### New content for the chart
Another nice feature to use with Helm charts is a `NOTES.txt` file in the `helm-chart/templates` folder. This file will be shown right after installation of the chart. And it's also available via the OpenShift user interface (UI) in Developer→Helm→Release Notes). Basically, you can enter your release notes there as a Markdown-formatted text file like the following. The nice thing is that you can insert named parameters defined in the `values.yaml` file:

```bash
# Release NOTES.txt of person-service helm chart
- Chart name: {{ .Chart.Name }}
- Chart description: {{ .Chart.Description }}
- Chart version: {{ .Chart.Version }}
- App version: {{ .Chart.AppVersion }}

## Version history
- 0.0.1 Initial release
- 0.0.2 release with some fixed bugs
- 0.0.3 release with image coming from quay.io and better parameter substitution
- 0.0.4 added NOTES.txt and a configmap
- 0.0.5 added a batch Job for post-install and post-upgrade
```

Parameters? Yes, of course. Sometimes you need to replace standard settings, as we did with the OpenShift Templates or Kustomize.

Let's have a closer look into the `values.yaml` file:
```yaml
deployment:
    image: quay.io/wpernath/person-service
    version: v1.0.0-test
    replicas: 2
    includeHealthChecks: false

config:
    greeting: 'We are on a newer version now!'
```

You are just defining your variables in the file. The following YAML configuration file illustrates how to insert the variables, through curly braces:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_GREETING: |-
    {{ .Values.config.greeting | default "Yeah, it's openshift time" }}
```

This `ConfigMap` defines a parameter named `APP_GREETING` with the content of the variable `.Values.config.greeting` (the initial period must be present). If this variable is not defined, the default will be used.

Helm also defines some [built-in objects](https://helm.sh/docs/chart_template_guide/builtin_objects/ "built-in objects") with predefined parameters:
- `.Release` can be used, after the Helm chart is installed in a Kubernetes environment. The parameter defines variables such as `.Name` and `.Namespace`.
- `.Capabilities` provides information about the Kubernetes cluster where the chart is installed.
- `.Chart` provides access to the content of the `Chart.yaml` file. Any data in that file is accessible.
- `.Files` provides access to all non-special files in a chart. You can't use this parameter to access template files, but you can use it to read and parse other files in your chart, for example to generate the contents of a `ConfigMap`.

Because [Helm's templating engine](https://helm.sh/docs/chart_template_guide/getting_started/ "Helm Templating Engine") is an implementation of the [templating engine](https://pkg.go.dev/text/template?utm_source=godoc) of the [Go](https://golang.org "Golang") programming language, you also have access to functions and flow control. For example, if you want to write only certain parts of the `deployment.yaml` file, you can do something like:

```yaml
{{- if .Values.deployment.includeHealthChecks }}
<do something here>
{{- end }}
```

Do you see the `-` at the beginning of the reference? This is necessary to get a well formatted YAML file after the template has been processed. If you would remove the `-`, you would have empty lines in the YAML.

### Debugging Templates
Typically, after you execute `helm install`, all the generated files are sent directly to Kubernetes. A couple commands help you debug your templates first.

- The `helm lint` command checks whether your chart is following best practices.

- The `helm install --dry-run --debug` command renders your files without sending them to Kubernetes.

### Defining a hook
It is often valuable to include sample data with an application for mocking and testing. But if you want to install a database with example data as part of your Helm chart, you need to find a way of initializing the database. This is where [Helm hooks](https://helm.sh/docs/topics/charts_hooks/ "Helm hooks") come into play.

Basically, a hook is just another Kubernetes resource (such as a job or a pod), which gets executed when a certain event is triggered. An event could take place at one of the following points:
- pre-install, pre-upgrade
- post-install, post-upgrade
- pre-delete, post-delete
- pre-rollback, post-rollback

The type of hook gets configured via the `helm.sh/hook` annotation. This annotation allows you to assigns weights to hooks in order to specify the order in which they run. Lower-numbered weights run before higher-number ones, for each type of event.

The following listing defines a new hook as a Kubernetes `Job` with `post-install` and `post-upgrade` triggers:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
    name: "{{ .Release.Name }}"
    labels:
        app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
        app.kubernetes.io/instance: {{ .Chart.Name | quote }}
        app.kubernetes.io/version: {{ .Chart.AppVersion }}
        "helm.sh/chart": "{{ .Chart.Name }}-{{ .Chart.Version }}"
    annotations:
        # This is what defines this resource as a hook. Without this line, the
        # job is considered part of the release.
        "helm.sh/hook": post-install,post-upgrade
        "helm.sh/hook-weight": "-5"
        "helm.sh/hook-delete-policy": before-hook-creation
spec:
    template:
        metadata:
            name: {{ .Chart.Name }}
            labels:
                "helm.sh/chart": "{{ .Chart.Name }}-{{ .Chart.Version }}"
        spec:
            restartPolicy: Never
            containers:
            - name: post-install-job
              image: "registry.access.redhat.com/ubi8/ubi-minimal:latest"
              command:
              - /bin/sh
              - -c
              - |
                echo "WELCOME TO '{{ .Chart.Name }}-{{ .Chart.Version }}' "
                echo "-------------------------------------------------"
                echo "Here we could now do initialization work."
                echo "Like filling our DB with some data or what's so ever"
                echo "..."

                sleep 10
```

As the example shows, you can also execute an arbitrary script as part of the install process.

### Subcharts and CRDs
In Helm, you can define subcharts. Whenever your chart gets installed, all dependent subcharts are installed as well. Just put required subcharts into the `helm/charts` folder.

This could be quite handy, if your application requires the installation of a database or other dependent components. 

Subcharts must be installable without the main chart. This means that each subchart has its own `values.yaml` file. You can override the values in the subchart's file within your main chart's `values.yaml`.

If your chart requires the installation of a custom resource definition (CRD)—for example, to install an Operator—simply put the CRD into the `helm/crds` folder of your chart. Keep in mind that Helm does *not* take care of deinstalling any CRDs if you want to deinstall your chart. So installing CRDs with helm is a one-shot operation.

### Summary of Helm charts
Creating a Helm chart is quite easy and mostly self-explaining. Features such as hooks help you do some initialization after installation.

If Helm charts do so much, so well, why would you need another package format? Well, let's next have a look at Operators.


## Kubernetes Operators
Our simple Quarkus application makes few demands of an administrator or of its hosting environment, whether plain Kubernetes or OpenShift. The application is a stateless, web-based application that doesn't require any special treatment by an administrator.

If you use the Helm chart to install the application (or even install it manually into OpenShift via `oc apply -f`), Kubernetes understands how to manage it out of the box pretty well. Kubernetes's control loop mechanism knows what the desired state of the application is (based on its various YAML files) and compares the desired state continually with the application's current state. Any deviations from the desired state are fixed automatically. For instance, if a pod has just died, Kubernetes takes care of restarting it. If you have uploaded a new version of the image, Kubernetes re-instantiates the whole application with the new image.

That's pretty easy.

But what happens if your application requires some complex integrations into other applications not built for Kubernetes, such as a database? Or if you want to back up and restore stateful data? Or need a clustered database? Such requirements typically require the special know-how of an administrator.

A Kubernetes Operator embodies those administrative instructions. It creates a package that contains, in addition to the information Kubernetes needs to deploy your application, the know-how of an administrator to maintain the complex, stateful part of the application.

Of course, this makes an Operator way more complex than a Helm chart, because all the logic needs to be implemented before putting it into the Operator. There are officially three ways to implement an Operator:

- Create it from Ansible.
- Create it from Helm chart.
- Develop everything in Go.

Unofficially (not supported right now), you can also implement the Operator's logic with any programming language, such as [Java via a Quarkus extension](https://github.com/quarkiverse/quarkus-operator-sdk "Quarkus Operator SDK Extension").

An Operator creates, watches, and maintains CRDs. This basically means that it provides new API resources to the cluster, such like a `Route` or `BuildConfig`. Whenever someone creates a new resource via `oc apply` based on the CRD, the Operator knows what to do. All the logic behind that mechanism is handled by the Operator. It makes extensively use of the Kubernetes API (just think about the work necessary to set up a clustered database or to back up and restore the persistent volume of a database).

If you need to have full control over everything, you have to create the Operator with Go or (unofficially) with Java. Otherwise, you can make use of an Ansible-based or Helm-based Operator. The Operator SDK and the base packages in Ansible and Helm take care of the Kubernetes API calls. So you don't have to learn Go now in order to build your first Operator.

### Creating a Helm-based Operator
To create an Operator, you need to [install the Operator-SDK](https://sdk.operatorframework.io/docs/installation/ "Install Operator-SDK"). On macOS, you can simply execute `brew install operator-sdk`.

#### Generating the project structure
In this example, we'll create an Operator based on the Helm chart created in earlier in the chapter:

```bash
$ mkdir kube-operator
$ cd kube-operator
$ operator-sdk init \
   --plugins=helm --helm-chart=../helm-chart \
   --domain wanja.org --group charts \
   --kind PersonService \
   --project-name person-service-operator
Writing kustomize manifests for you to edit...
Creating the API:
$ operator-sdk create api --group charts --kind PersonService --helm-chart ../helm-chart
Writing kustomize manifests for you to edit...
Created helm-charts/person-service
Generating RBAC rules
I1027 13:19:52.883020   72377 request.go:665] Waited for 1.017668738s due to client-side throttling, not priority and fairness, request: GET:https://api.art6.ocp.lan:6443/apis/controlplane.operator.openshift.io/v1alpha1?timeout=32s
WARN[0002] The RBAC rules generated in config/rbac/role.yaml are based on the chart's default manifest. Some rules may be missing for resources that are only enabled with custom values, and some existing rules may be overly broad. Double check the rules generated in config/rbac/role.yaml to ensure they meet the Operator's permission requirements.
```

These commands initialize the project for the Operator based on the chart found in the `../helm-chart` folder. A `PersonService` CRD should also have been created in `config/crd/bases`. The following listing shows the complete directory structure generated by the commands:

```bash
$ tree
.
├── Dockerfile
├── Makefile
├── PROJECT
├── config
│   ├── crd
│   │   ├── bases
│   │   │   └── charts.wanja.org_personservices.yaml
│   │   └── kustomization.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   └── manager_config_patch.yaml
│   ├── manager
│   │   ├── controller_manager_config.yaml
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── manifests
│   │   └── kustomization.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── personservice_editor_role.yaml
│   │   ├── personservice_viewer_role.yaml
│   │   ├── role.yaml
│   │   ├── role_binding.yaml
│   │   └── service_account.yaml
│   ├── samples
│   │   ├── charts_v1alpha1_personservice.yaml
│   │   └── kustomization.yaml
│   └── scorecard
│       ├── bases
│       │   └── config.yaml
│       ├── kustomization.yaml
│       └── patches
│           ├── basic.config.yaml
│           └── olm.config.yaml
├── helm-charts
│   └── person-service
│       ├── Chart.yaml
│       ├── templates
│       │   ├── NOTES.txt
│       │   ├── config-map.yaml
│       │   ├── deployment.yaml
│       │   ├── post-install-hook.yaml
│       │   ├── route.yaml
│       │   └── service.yaml
│       └── values.yaml
└── watches.yaml

15 directories, 41 files
```

The `watches.yaml` file is used by the Helm-based Operator to watch changes on the API. So whenever you create a new resource based on the CRD, the underlying logic knows what to do.

Now have a look at the `Makefile`. There are three parameters in it that you should change:

- `VERSION`: Whenever you change something in the project file (and have running instances of the Operator somewhere), increment the number.
- `IMAGE_TAG_BASE`: This is the base name of the images that the makefile produces. Change the name to something like `quay.io/wpernath/person-service-operator`.
- `IMG`: This is the name of the image within our Helm-based Operator. Change the name to something like `$(IMAGE_TAG_BASE):$(VERSION)`.

#### Building the Docker image
Now build and push the Docker image of your Operator:

```bash
$ make docker-build
docker build -t quay.io/wpernath/person-service-operator:0.0.1 .
[+] Building 6.3s (9/9) FINISHED

[...]

 => [2/4] COPY watches.yaml /opt/helm/watches.yaml                                                                                                          0.1s
 => [3/4] COPY helm-charts  /opt/helm/helm-charts                                                                                                           0.0s
 => [4/4] WORKDIR /opt/helm                                                                                                                                 0.0s
 => exporting to image                                                                                                                                      0.0s
 => => exporting layers                                                                                                                                     0.0s
 => => writing image sha256:b5f909021fe22d666182309e3f30c418c80d3319b4a834c5427c5a7a71a42edc                                                                0.0s
 => => naming to quay.io/wpernath/person-service-operator:0.0.1

$ make docker-push
docker push quay.io/wpernath/person-service-operator:0.0.1
The push refers to repository [quay.io/wpernath/person-service-operator]
5f70bf18a086: Pushed
fe540cb9bcd7: Pushed
ddaee5130ba6: Pushed
4ea9a10139f9: Mounted from operator-framework/helm-operator
5a5ce86c51f0: Mounted from operator-framework/helm-operator
3a40d0007ffb: Mounted from operator-framework/helm-operator
0b911edbb97f: Mounted from wpernath/person-service
54e42005468d: Mounted from wpernath/person-service
0.0.1: digest: sha256:acd3f89a7e0788b226d5016df765f61a5429cf5cef6118511a0910b9f4a04aaf size: 1984
```

In your repository on Quay.io, you now have a new image called `person-service-operator`. This image contains the logic to manage the Helm chart. The image exposes the CRD and the new Kubernetes API.

#### Testing your Operator
There are currently three different ways of run an Operator, listed in the official [SDK tutorial](https://sdk.operatorframework.io/docs/building-operators/helm/tutorial/ "SDK tutorial"). The easiest way to test your Operator is to run:

```bash
$ make deploy
cd config/manager && /usr/local/bin/kustomize edit set image controller=quay.io/wpernath/person-service-operator:0.0.1
/usr/local/bin/kustomize build config/default | kubectl apply -f -
namespace/person-service-operator-system created
customresourcedefinition.apiextensions.k8s.io/personservices.charts.wanja.org created
serviceaccount/person-service-operator-controller-manager created
role.rbac.authorization.k8s.io/person-service-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/person-service-operator-manager-role created
clusterrole.rbac.authorization.k8s.io/person-service-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/person-service-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/person-service-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/person-service-operator-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/person-service-operator-proxy-rolebinding created
configmap/person-service-operator-manager-config created
service/person-service-operator-controller-manager-metrics-service created
deployment.apps/person-service-operator-controller-manager created
```

This will create a `person-service-operator-system` namespace and install all the necessary files into Kubernetes. This namespace contains everything needed to handle the request to create a `PersonService`, but does not contain any instances of your newly created custom resource.

If you now want to see your `PersonService` in action, create a new namespace and apply a new instance of the `PersonService` through the following configuration file `config/samples/charts_v1alpha1_personservice`:

```yaml
apiVersion: charts.wanja.org/v1alpha1
kind: PersonService
metadata:
  name: my-person-service1
spec:
  # Default values copied from <project_dir>/helm-charts/person-service/values.yaml
  config:
    greeting: Hello from inside the operator!!!!
  deployment:
    image: quay.io/wpernath/person-service
    includeHealthChecks: false
    replicas: 1
    version: v1.0.0-test
```

The following commands create the service:

```bash
$ oc new-project book-operator
$ oc apply -f config/samples/charts_v1alpha1_personservice
personservice.charts.wanja.org/my-person-service1 configured
```

This will take a while. But after some time you will notice a change in the Topology view of OpenShift, showing that the Helm chart was deployed.

To delete everything, delete all instances of the CRD you've created:

```bash
$ oc delete personservice/my-person-service1
$ make undeploy
```

#### Building and running the Operator bundle image
In order to release your Operator, you have to create an Operator bundle. This bundle is an image with metadata and manifests use by the Operator Lifecycle Manager (OLM), which takes care of every Operator deployed on Kubernetes. To create the bundle, enter:

```bash
$ make bundle
operator-sdk generate kustomize manifests -q
cd config/manager && /usr/local/bin/kustomize edit set image controller=quay.io/wpernath/person-service-operator:0.0.3
/usr/local/bin/kustomize build config/manifests | operator-sdk generate bundle -q --overwrite --version 0.0.3
INFO[0000] Creating bundle.Dockerfile
INFO[0000] Creating bundle/metadata/annotations.yaml
INFO[0000] Bundle metadata generated suceessfully
operator-sdk bundle validate ./bundle
INFO[0000] All validation tests have completed successfully
```

The bundle generator asks you a few questions to configure the bundle. Have a look at Figure 4 for the output.

![Image 4: Building the bundle](Bildschirmfoto%202021-05-20%20um%2022.06.03.png)
 

Run the generator every time you change the `VERSION` field in the `Makefile`:

```bash
$ make bundle-build bundle-push
docker build -f bundle.Dockerfile -t quay.io/wpernath/person-service-operator-bundle:v0.0.3 .
[+] Building 0.2s (7/7) FINISHED
 => [internal] load build definition from bundle.Dockerfile                                                                                                                   0.0s
 => => transferring dockerfile: 987B                                                                                                                                          0.0s
 => [internal] load .dockerignore                                                                                                                                             0.0s
 => => transferring context: 2B                                                                                                                                               0.0s
 => [internal] load build context                                                                                                                                             0.0s
 => => transferring context: 11.89kB                                                                                                                                          0.0s
 => [1/3] COPY bundle/manifests /manifests/                                                                                                                                   0.0s
 => [2/3] COPY bundle/metadata /metadata/                                                                                                                                     0.0s
 => [3/3] COPY bundle/tests/scorecard /tests/scorecard/                                                                                                                       0.0s
 => exporting to image                                                                                                                                                        0.0s
 => => exporting layers                                                                                                                                                       0.0s
 => => writing image sha256:7f712b903cbdbf12b6f34189cdbca404813ade4d7681d3d051fb7f6b2e12d0f5                                                                                  0.0s
 => => naming to quay.io/wpernath/person-service-operator-bundle:v0.0.3                                                                                                       0.0s
/Library/Developer/CommandLineTools/usr/bin/make docker-push IMG=quay.io/wpernath/person-service-operator-bundle:v0.0.3
docker push quay.io/wpernath/person-service-operator-bundle:v0.0.3
The push refers to repository [quay.io/wpernath/person-service-operator-bundle]
413502b46f1b: Pushed
25ab801d3fd7: Pushed
f01cda5511c8: Pushed
v0.0.3: digest: sha256:780980b7b7df76faf7be01a7aebcdaaabc4b06dc85cc673f69ab75b58c7dca0c size: 939
```

The bundle image gets pushed to Quay.io as `quay.io/wpernath/person-service-operator-bundle`. Finally, to install the Operator, enter:
```bash
$ operator-sdk run bundle quay.io/wpernath/person-service-operator-bundle:v0.0.3
```

This command installs the Operator into OpenShift and registers it with the Operator Lifecycle Manager (OLM). After this, you'll be able to watch and manage your Operator via the OpenShift UI, just like the Operators that come with OpenShift (Figure 5).
![Image 5: The installed Operator in OpenShift UI](Bildschirmfoto%202021-10-27%20um%2018.49.28.png)

You can create a new instance of the service in the UI by clicking Installed Operators→PersonService→Create PersonService, or by executing the following at the command line:

```bash
$ oc apply -f config/samples/charts_v1alpha1_personservice
personservice.charts.wanja.org/my-person-service1 configured
```

Shortly after requesting the new instance, you should see the deployed Helm chart of your `PersonService`.

### Cleaning up
If you want to get rid of this Operator, just enter:

```bash
$ operator-sdk cleanup person-service-operator --delete-all
```

The `cleanup` command needs the project name, which you can find in the `PROJECT` file.

### Summary of Operators
Creating an Operator just as a replacement for a Helm chart does not really make sense, because Operators are much more complex to create and maintain. And what we've done so far is just the tip of the iceberg. We haven't really touched the Kubernetes API.

However, as soon as you need more influence over the creation and maintenance of your application and its associated resources, think about building an Operator. Fortunately, the Operator SDK and the documentation help you with the first steps.

## Summary
This chapter has described how to build images. You've learned more about the various command-line tools (`skopeo`, `podman`, `buildah` etc.). You also saw how to create a Helm Chart and a Kubernetes Operator. And you should be able to decide when to use each tool.

The next chapter of this book will talk about Tekton pipelines as a form of internal distribution.