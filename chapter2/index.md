# Automated Application Packaging and Distribution With OpenShift - Basic Development Principles - Part 1/4
This is part one of a four parts article series I have written to discuss tools for automated application packaging and distribution in the world of containers.

In this part, I am discussing the basics of app deployment automation. I am going through an example, which you can find here 
[https://github.com/wpernath/quarkus-simple.git](https://github.com/wpernath/quarkus-simple.git "Simple Quarkus Example") 

First, we are going to deploy this example and let OpenShift build it. Then we are analyzing which files were generated, how to extract them out of OpenShift and how to modify them to easily recreate your app in another namespace. Then we are going to discuss OpenShift Templates and finally Kustomize. 

In the next part of this article series, we are going to discuss two ways of packaging your application via Helm Charts and Operators. 

## Introduction and Motivation
As someone with a long history of developing software I do like containers and Kubernetes a lot, as those technologies will help me to increase my own productivity as I do not have to wait too much anymore to get what I need (a remote testing system, for example) from the ops departments. 

On the other side, writing applications, especially micro services for Kubernetes, could easily become quite complex, because I suddenly also have to maintain artifacts which do not necessarily belong to me:

- ConfigMaps and secrets (well, I somehow have to store my app config anyway)
- Deployment.yaml
- Service.yaml
- Ingress.yaml or Route.yaml
- PersistentVolumeClaim.yaml
- etc.

In native Kubernetes, I have to take care of creating and maintaining those artifacts. Thanks to the Source-to-image concept of OpenShift, I don’t have to worry about most of those artifacts. They will be generated for me. 

```bash
$> oc new-project article-dev
$> oc new-app java:openjdk-11-ubi8~https://github.com/wpernath/quarkus-simple.git --name=simple --build-env MAVEN_MIRROR_URL=<your maven repo on openshift>
$> oc expose service/simple
route.route.openshift.io/simple exposed
```

This will create a new project „article-dev“ in OpenShift. It will then create a new app based on the java builder image „openjdk-11-ubi8“ and provides it with the source code coming from GitHub. If you don’t have a local Maven mirror, ignore the —build-env parameter.

Security settings, deployment, image, route and service will be generated for me (and also some OpenShift specific files, like ImageStream and / or DeploymentConfig). So I am able to fully focus on app development now. 

But what if I want to deploy my app from DEV stage over to the TEST stage? And how to automate this in tools like Jenkins or Tekton?

What exactly are the artifacts I have to care about (it’s not just the image). And how to pull them over into the next stages? And how do they differ from my version?

And what should I do, if I want to release my application? Should I use Helm? Should I use just an image repository like https://quay.io/ or the DockerHub? Or should I also invest in Operators? And when should I use what? And why? 

Those are exactly the questions, I am trying to answer with this series of blog posts.  

## Basic Kubernetes Files
So what are the necessary artifacts of an OpenShift app deployment? 

**Deployment:** A deployment connects the image with a container and provides various runtime informations like environment variables, startup scripts, config map references etc. It also contains a definition of used ports 
**DeploymentConfig:** This is OpenShift specific config file, which has mainly  the same functionality as a **Deployment**. If you start today with your OpenShift tour, use **Deployment** instead of this one. 
**Service:** A service contains the runtime information which kubernetes needs to load balance your application over different instances (pods)
**Route:** A route defines the external URL and where to route requests to.
** ConfigMap:** ConfigMaps contain - well - configurations for the app.
**Secret:** Like a ConfigMap, a Secret contains encrypted (well) password informations.

Once those files are automatically generated, you can get them by using kubectl or oc:

```bash
$> oc get deployment
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
simple   1/1     1            1           17m
```

If you’re specifying „-o yaml“ option, you get the complete descriptor:
```bash
$> oc get deployment simple -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    alpha.image.policy.openshift.io/resolve-names: '*'
    app.openshift.io/vcs-ref: ""
    app.openshift.io/vcs-uri: https://github.com/wpernath/quarkus-simple.git
    deployment.kubernetes.io/revision: "2"
    image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"simple:latest","namespace":"wpernath-article"},"fieldPath":"spec.template.spec.containers[?(@.name==\"simple\")].image","pause":"false"}]'
    openshift.io/generated-by: OpenShiftWebConsole
  creationTimestamp: "2021-04-20T08:36:11Z"
  generation: 2
  labels:
    app: simple
    app.kubernetes.io/component: simple
    app.kubernetes.io/instance: simple
    app.kubernetes.io/name: java
    app.kubernetes.io/part-of: quarkus
    app.openshift.io/runtime: java
    app.openshift.io/runtime-version: openjdk-11-ubi8
[...]
```

Just pipe the output into a new .yaml file and you’re done. You can directly use this file to create your app in a new namespace (except for the image section). But of course you would like to flatten the file a bit. There is a lot of garbage in there you don’t need. For example the „managedFileds“ section. Or the metadata section in the beginning. Just strip it down and add the file to your git repository.
 ![Image 1: A stripped down route.yaml file after exporting it](Bildschirmfoto%202021-04-21%20um%2008.21.23.png)
Just do the same with Route and Service and that’s all for now. You’re now able to create your app in a new namespace by simply doing:

```bash
$> oc new-project article-test
$> oc policy add-role-to-user system:image-puller system:serviceaccount:article-test:default --namespace=article-dev
$> oc apply -f kubernetes-files/service.yaml
$> oc apply -f kubernetes-files/deployment.yaml
$> oc apply -f kubernetes-files/route.yaml
```

The command oc policy is necessary to let the namespace „article-test“ access the image in the namespace „article-dev“. Otherwise you’d get an error message in OpenShift, saying that the image was not found.

So that’s one way of getting required files. Of course, if you have more things in your application, you need to export those files as well. If you have PersistenceVolumeClaims or ConfigMaps or Secrets, you need to export them as well.

And especially for the Deployment and the Route, you have to manually change fields. For example, it does not make sense to use the lastest image from the Dev namespace in the Test one. You’d always have the same version of your application. This means, you have to change the image in the Deployment on every stage you’re using. This is where tools like OpenShift Templates or Kustomize come into play. 

## OpenShift Templates
OpenShift Templates are an easy way to create one file out of the required configuration files and add parameters in there. 

Unfortunately - as the name say -, they are OpenShift specific. But it is quite easy to create a template file out of the files, we’ve exported and modified above. 
![Image 2: An OpenShift Template file](Bildschirmfoto%202021-04-21%20um%2009.03.54.png)

To create a new template file, simply open your preferred editor and create a file called template.yaml. The header of that file should look like this:

```bash
apiVersion: template.openshift.io/v1
kind: Template
name: app-template
metadata:
  name: app-template
  annotation:
    tags: java
    iconClass: icon-rh-openjdk
    openshift.io/display-name: The APPLICATION template    
    description: This Template creates the APPLICATION
objects:
```

We are then adding our three files (route.yaml, deployment.yaml and service.yaml) into this file right under the objects tag. 

```bash
  - apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: ${APPLICATION_NAME}
      name: ${APPLICATION_NAME}
    spec:
      ports:
      - name: 8080-tcp
        port: 8080
        protocol: TCP
      selector:
        app: ${APPLICATION_NAME}
```

As you can see, we are already using parameters here. Of course we have to define the parameters in the „parameters“-section of the yaml file:

```bash
parameters:
- name: APPLICATION_NAME
  description: The name of the application you'd like to create
  displayName: Application Name
  required: true
  value: simple
```

The reason why the OpenShift Template definition allows us to define properties like description and displayName is that a template, once instantiated in an OpenShift namespace, can be used to create applications from within the UI. 

```bash
$> oc new-project article-template
$> oc apply -f kubernetes-files/template.yaml
template.template.openshift.io/app-template created
```

Just open the OpenShift web console now, change the project and click on „+Add“ and choose the Developer Catalog. You should be able to find a template called „app-template“. This is the one we’ve created. 
![Image 3: The Developer Catalog after adding the template ](Bildschirmfoto%202021-04-21%20um%2009.25.13.png)
Instantiate the template and fill in the required fields. 
![Image 4: Template instantiation with required fields](Bildschirmfoto%202021-04-21%20um%2009.28.55.png)
Then click on create and after a short while, you should see the application’s deployment progressing. Once it is done, you should be able to access the route. 

With this command you’re able to create your application based on the template without the UI:

```bash
$> oc new-app app-template -p APPLICATION_NAME=simplecli -p HOST_NAME=simplecli-article-template.apps.example.com
--> Deploying template "article-template/app-template" to project article-template

     * With parameters:
        * Application Name=simplecli
        * Image Name=quay.io/wpernath/simple-quarkus
        * Image Tag=latest
        * Host Name= simplecli-article-template.apps.example.com

--> Creating resources ...
    service "simplecli" created
    deployment.apps "simplecli" created
    route.route.openshift.io "simplecli" created
--> Success
    Access your application via route 'simplecli-article-template.apps.example.com'
    Run 'oc status' to view your app.
```


### Summary of using OpenShift Templates
Creating and maintaining an OpenShift Template is not that hard. The way of specifying parameters is self explaining. And I personally like the deep integration into OpenShift’s developer console and the `oc` command. 

Unfortunately, they are OpenShift only. Which means, if you are using a local Kubernetes installation and a production OpenShift version, it’s not easily possible to reuse them. But if your complete development and production environments are based on OpenShift, you should give it a try. 

I personally would prefer Templates if I should take care of standardization of the development process. I could create a template of a standard application (including BuildConfig etc.), import it into the global `openshift` project so that all users are able to reuse my base work. — Just like the other OpenShift Templates shipped with any OpenShift installation.  

## Kustomize
Kustomize is not a templating engine. Kustomize is using the fact, that only a few fields have to be changed from stage to stage. Which means, you’re using a base set of files (deployment, service, route etc.) and for each stage you’re creating files just with the changes. The patch mechanism of Kustomize takes care of merging the files together. 

This is very handy if you don’t want to learn just another templating engine. Or if you don’t want to maintain a file which could easily contain thousands of lines of informations (OpenShift Templates). 

Kustomize was originally founded by Google and is now a subproject of Kubernetes. `kubectl` and `oc` have everything necessary build-in.

Let’s have a look in how Kustomize works:

```bash
tree kustomize
kustomize
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── route.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   └── route.yaml
    ├── prod
    └── stage
        ├── deployment.yaml
        ├── kustomization.yaml
        └── route.yaml

5 directories, 10 files
```

As you can see, there are a set of base files and an overlays directory. Base defines all resources which Kubernetes / OpenShift needs in order to deploy your application (they are all well known from the other chapters above). 

Just `kustomization.yaml` is new. Let’s have a look at this file:
![Image 5: Structure of kustomization.yaml ](Bildschirmfoto%202021-04-21%20um%2016.52.32.png)

As you can see, this file defines its resources (deployment, service and route) but also adds a section called „commonLabels“. Those labels will be applied to all resources generated by Kustomize. 

With:

```bash
$> oc new-project article-kustomize
$> oc apply -k overlays/dev
service/simple created
deployment.apps/simple created
route.route.openshift.io/simple created
```

We are able to process all files and deploy our application. If you’re installing the Kustomize cli tool (for example with `brew install kustomize` on macOS), you’re able to debug the output:
![Image 6: Output of kustomize tool](Bildschirmfoto%202021-04-21%20um%2017.00.07.png)

A big benefit of using Kustomize is that you’re maintaining only the differences between each stage, meaning that the overlay files are quite small and clear. If a file does not change between a stage and the other, it does not need to be duplicated. 

With kustomization fields like „commonLabels“ or „commonAnnotations“, you can specify labels or annotations which you would like to have in every metadata section of every generated file. „namePrefix“ will be used to specify a prefix on every name tag. 

```bash
$> kustomize build kustomize/overlays/stage
```

This command shows you how Kustomize will merge the files together for the stage overlay. As you can see, all files have „staging“ as name prefix and additionally we have a new commonLabel (variant: staging) and an annotation (note: we are on staging now). 
 ![Image 7: Output of kustomize tool for stage overlay](Bildschirmfoto%202021-04-22%20um%2008.11.36.png)

But we still have „app“ and „org“ labels specified. 

```bash
$> oc apply -k kustomize/overlays/stage
```

### More sophisticated examples?
Instead of using „patchStrategicMerge“ files, one could just maintain a `kustomization.yaml` file with everything in there.

![Image 8: Using inline patches](Bildschirmfoto%202021-04-24%20um%2007.20.44.png)

There are specific fields in newer versions of Kustomize which helps you to maintain your stages. For example, if it’s just about changing the tag of the target image, you could simply use the „images“ field array specifier (have a look at Image 8). 

With the „patches“ field array, you can issue a patch on a list of target files, like replacing the target host name of the route or adding health checks for the application in the Deployment (see Image 9 and 10). 

![Image 9: Adding health checks to the Deployment](Bildschirmfoto%202021-04-24%20um%2007.27.29.png)

![Image 10: The patch to add health checks](Bildschirmfoto%202021-04-24%20um%2007.28.02.png)

You can even generate the ConfigMap based on fixed parameters or properties files. 

Unfortunately, it seems that `oc apply -k` does not contain recent Kustomize features. So if you want to use them, you need to use the `kustomize` cli tool and pipe the output to `oc apply -f`.

```bash
$> kustomize build kustomize_ext/overlays/stage | oc apply -f -
```

 For more information and more sophisticated examples have a look at the official home page: [https://kustomize.io/](https://kustomize.io/)

### Summary of using Kustomize
Using Kustomize is quite easy and straight forward. You don’t really have to learn a templating DSL, you just need to understand patch and merge and how it works. Kustomize makes it easy for you as a CI/CD guy to separate the configuration of your application for every stage. As Kustomize is also a Kubernetes subproject and tightly integrated into Kubernetes’ tools, you don’t have to worry that Kustomize would suddenly disappear.

ArgoCD has build-in support for Kustomize as well, so that if you’re doing CD with Argo, you have automatically support for this as well. 

## Summary
In this article we have learned how to build an application using OpenShift’s source-to-image and we’ve learned OpenShift Templates and Kustomize. This is the base technology for automation of application deployment and packaging. 

Now we have an understanding of which artifacts need to be taken into account when we want to release our application and how to modify those to make sure, the new environment is capable of handling our app. 

In the next article, which should be posted end of May 2021, we are using what we’ve learned here to build a package with Helm or Operators. 

