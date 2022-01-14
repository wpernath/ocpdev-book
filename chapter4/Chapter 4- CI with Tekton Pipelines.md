# Chapter 4: CI with Tekton Pipelines
Chapters 2 and 3 of this book discussed the basics of application packaging with Kustomize, Helm Charts, and Operators. They also discussed how to handle images and all the metadata required for working with Kubernetes.

This chapter discusses how to integrate complex tasks, such as building and deploying applications into Kubernetes using [Tekton][1]. Continuous development and continuous integration (CD/CI) are represented in Tekton as *pipelines* that combine all the steps you need to accomplish what you want. And Tekton makes it easy to write a general pipeline that you can adapt to many related tasks.

## Tekton and OpenShift Pipelines
[Tekton][2] is an open source framework to create pipelines for Kubernetes and the cloud. This means that there is no central tool you need to maintain such as Jenkins. You just have to install a Kubernetes Operator into your Kubernetes cluster to provide some custom resource definitions (CRDs). Based on those CRDs, you can create tasks and pipelines to compile, test, deploy, and maintain your application.

[OpenShift Pipelines][3] is based on Tekton and adds a nice GUI to the OpenShift developer console. The Pipelines Operator is free to use for every OpenShift user.

### Tekton Concepts
Tekton has numerous objects, but the architecture is quite easy to understand. The key concepts are:

- Step: A process that runs in its own container and can execute whatever the container image provides. A step does not stand on its own, but must be embedded in a task.
- Task: A set of steps running in separate containers (known as "pods" in Kubernetes). A task could be, for example, a compilation process using Maven. One step would be to check the Maven `settings.xml` file. The second step could be to execute the Maven goals (compile, package etc.).
- Pipeline: A set of tasks that are executed either in parallel or (in a simpler case) one after another. A pipeline can be customized through *parameters*.
- PipelineRun: A collection of parameters to submit to a pipeline. For instance, a build-and-deploy pipeline might refer to a PipelineRun that contains technical input (for example, ConfigMaps and PersistentVolumeClaims) as well as non-technical parameters (for example, the URL for the Git repository to clone, the name of the target image, etc.)

Internally, Tekton creates a TaskRun object for each task it finds in a PipelineRun.

To summarize: A pipeline contains a list of tasks, each of which contain a list of steps. One of the benefits of Tekton is that tasks and pipelines can be shared with other people, because a pipeline just specifies what to do in a given order. So if most of your projects have a similar pipeline, share and reuse it.

### Installing the tkn command-line tool
Tekton comes with a command-line tool called `tkn`, which you can easily install on macOS by issuing:

```bash
$ brew install tektoncd-cli
```

Please check the official homepage of Tekton to see how to install the tool on other operating systems.

### Installing OpenShift Pipelines on Red Hat OpenShift

The process in this chapter requires version 1.4.1 or higher of the OpenShift Pipelines Operator. To install that version, you also need a recent 4.7 OpenShift cluster, which you could install for example via [CodeReady Containers][4]). Without these tools, you won’t have access to workspaces (which you need to define).

To install OpenShift Pipelines, you must be cluster-admin. Go to the OperatorHub, search for "pipelines," and click the **Install** button. There is nothing more to do for now, as the Operator maintains everything for you (Figure 1).

![Image 1: Using the OpenShift UI to install OpenShift Pipelines operator][image-1]

After a while you’ll notice a new GUI entry in both the Administrator and the Developer UI (Figure 2).
![Image 2: New UI entries in the OpenShift GUI after you've installed the Pipelines Operator][image-2]

## This chapter's example: Create a pipeline for quarkus-simple
For our well-known [person-service][5], we are going to create a Tekton pipeline for a simple deployment task. The pipeline compiles the source, creates a Docker image based on [Jib][6], pushes the image to [Quay.io][7] and uses [kustomize][8] to apply that image to an OpenShift project called `book-tekton`.

Sounds easy?

It is. Well, mostly.

First of all, why are we going to use Jib for building the container image? Well, that’s easily explained: Right now, there are three different container image build strategies available with Quarkus:
- Docker
- Source-to-Image (S2I)
- Jib
The Docker strategy uses the `docker` binary to build the container image. But the `docker` binary is not available inside a Kubernetes cluster (as mentioned in Chapter 3), because Docker is too heavyweight and requires root privileges to run the daemon.

S2I requires creating OpenShift specific BuildConfigs, DeploymentConfigs, and ImageStreams, which are not available in vanilla Kubernetes clusters.

In order to stay vendor-independent, we have to use Jib for this use case.

Of course, you could also use other tools to create your container image inside Kubernetes. But in order to keep this Tekton example clean and simple, we are reusing what Quarkus provides. So we are able to simply set a few Quarkus properties in `application.properties` to define how Quarkus should package the application. Now we are able to use exactly *one* Tekton task to compile, package, and push the application to an external registry.

**NOTE**: Make sure that your Quarkus application is using the required Quarkus extension called `container-image-jib`. If your `pom.xml` file does not include the `quarkus-container-image-jib` dependency, add it by executing:

```bash
$ mvn quarkus:add-extension -Dextensions="container-image-jib"
[INFO] Scanning for projects...
[INFO]
[INFO] -------------------< org.wanja.book:person-service >--------------------
[INFO] Building person-service 1.0.0
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- quarkus-maven-plugin:2.4.2.Final:add-extension (default-cli) @ person-service ---
[INFO] Looking for the newly published extensions in registry.quarkus.io
[INFO] [SUCCESS] ✅  Extension io.quarkus:quarkus-container-image-jib has been installed
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.817 s
[INFO] Finished at: 2021-11-22T09:34:05+01:00
[INFO] ------------------------------------------------------------------------
```

Then have a look at Figure 3 to see what properties need to be set to let Quarkus build, package, and push the image. Basically, the following properties need to be set in `application.properties`:
![Image 3: Application properties of the person-service][image-3]

1. `quarkus.container-image.build`: Set this to `true` to ensure that a `mvn package` command a container image.
2. `quarkus.container-image.push`: This is optional and required only if you want to push the image directly to the registry. I don't intend to do so, so I set the value to `false`.
3. `quarkus.container-image.builder`: Selects the method of building the container image. We set the value to `jib` to use [Jib][9].
4. `quarkus.container-image.image`: Set this to the complete name of the image to be built, including the domain name.

Now check out the [source code][10], have a look at `person-service/src/main/resources/application.properties`, change the image property to meet your needs, and issue:

```bash
$ mvn clean package -DskipTests
```

This command compiles the sources and builds the container image. If you want to push the resulting image to your registry, simply call:

```bash
$ mvn package -DskipTests -Dquarkus.container-image.push=true
```

After a while, Quarkus will generate and push your image to your registry. In my case, it’s `quay.io/wpernath/person-service?`.

### Inventory check: What do we need?
To create our use case, we need to have the following tasks available:

- `git`: To fetch the source from GitHub.
- `maven`: To reuse most of what Quarkus provides.
- `kustomize`: To change our Deployment to point to the new image.
- OpenShift client: To apply the changes we’ve made in the previous steps.

Let’s log into our OpenShift cluster, create a new project and list all the available ClusterTasks:

```bash
$ oc login .....
$ oc new-project book-tekton
$ tkn ct list
```

Figure 4 shows all the available ClusterTasks created after you install the OpenShift Pipelines Operator. It seems we have most of what we need:
- `git-clone`
- `maven`
- `openshift-client`

![Image 4: Available ClusterTasks in OpenShift after installation of pipelines operator][image-4]

We are just missing the `kustomize` task. We'll create one later. We first want to take care of the rest of the tasks.

Wait — what is the difference between a task and a ClusterTask? The answer is easy: A ClusterTask is available globally in all projects, whereas a task is available only locally per project and must be installed into each project, where you want to use it.

### Analyzing the necessary tasks
If you want to have a look at the structure of a task, you can easily do so by executing the following command:

```bash
$ tkn ct describe <task-name>
```

The output explains all the parameters of the task, together with other necessary information such as its inputs and outputs.

By specifying the `-o yaml` parameter, you can view the YAML source definition of the task.

The `git-clone` task allows a large number of parameters, but most of them are optional. We just have to specify `git-url` and `git-revision`. And we have to specify a workspace for the task.

### What are workspaces?
Remember that Tekton is running each and every task (and all steps inside a task) as a separate pod. If the application running on the pod writes to some random folder, nothing gets really stored. So if we want (and yes, we do want) the pipeline to be able to read and write data on each step, to be shared with other steps, we have to find a way to do that.

This is what workspaces are for. They could be a persistent volume claim, a config map etc. A task which requires either something to store data to (like git-clone) or needs to have access to data coming from a previous step (such as Maven), defines a workspace. If the task is embedded into a pipeline, it will also be defined there. The PipelineRun (or in case of a single running task, the TaskRun) finally creates the mapping between the defined workspace and a corresponding storage.

In our example we need two workspaces:
- A PersistentVolumeClaim (PVC) where the git-clone task is cloning the source code to and from where the Maven task is compiling the source
- A ConfigMap with the `maven-settings` file you need in your environment

## Ways of building the pipeline
Once you know what tasks you need in order to build your pipeline, you can start creating it. There are two ways of doing so:
- Build your pipeline via a code editor as a YAML file.
- Build your pipeline in the OpenShift Developer Console.

As a first try, I recommend building the pipeline via the graphical Developer Console of OpenShift. Then export and see it what it looks like.

### Building the pipeline via OpenShift Developer Console
Remember that you should have at least version 1.4.1 of the OpenShift Pipelines Operator installed.

![Image 5: Pipeline Builder][image-5]

You have to provide parameters to each task, and link the required workspaces to the tasks. You can easily do that by using the GUI.

![Image 6:Linking the workspaces from Pipeline to task][image-6]

We want to use the `maven` task twice, using the `package` goal:
- To simply compile the source code.
- To execute the `package` goal with the following parameters that [instruct quarkus to build and push the image][11]:
	- `-Dquarkus.container-image.push=true`
	- `-Dquarkus.container-image.builder=jib`
	- `-Dquarkus.container-image.image=$(params.image-name)`
	- `-Dquarkus.container-image.username=$(params.image-username)`
	- `-Dquarkus.container-image.password=$(params.image-password)`

Once you’ve done all that and have clicked on the **Save** button, you’re able to export the YAML file by executing:

```bash
$ oc get pipeline/build-and-push-image -o yaml > tekton/pipelines/build-and-push-image.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-push-image
spec:
  params:
  - default: https://github.com/wpernath/book-example.git
    description: the URL of the Git repoisotry to build
    name: git-url
    type: string
....
```

You can easily re-import the pipeline file by executing:

```bash
$ oc apply -f tekton/pipelines/build-and-push-image.yaml
```

### Placement of task parameters
One of the goals of Tekton has always been to provide tasks and pipelines that are as reusable as possible. This means making each task as general-purpose as possible.

If you’re providing the necessary parameters directly to each task, you might repeat the settings over and over again (for example, in our case, we are using the Maven task for compiling, packaging, image generation, and pushing). In this case it makes sense to draw the parameters out of the specification of each task. Put them on the pipeline level under an property called `params` (Figure 7) and reference them inside the corresponding task by specifying them by their name in the syntax `$(params.parameter-name)`.

![Image 7: The pipeline file as exported after using the GUI with top level parameters][image-7]

### Creating a new task: kustomize
Remember that our default OpenShift Pipelines Operator installation didn't include Kustomize. Because we want to use it to apply the new image to our Deployment, we have to look for a proper task in [Tekton Hub][12]. Unfortunately, there doesn’t seem to be one available, so we have to create our own.

For this, we first need to have a proper image which contains the `kustomize` executable. The Dockerfile for this project is available in the [kustomize-ubi repository on GitHub][13] and the image is available in [its repository on Quay.io][14].

Now let’s create a new Tekton task:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: kustomize
  labels:
    app.kubernetes.io/version: "0.4"
  annotations:
    tekton.dev/pipelines.minVersion: "0.12.1"
    tekton.dev/tags: build-tool
spec:
  description: >-
    This task can be used to execute kustomze build scripts and to apply the changes via oc apply -f
  workspaces:
    - name: source
      description: The workspace holding the cloned and compiled quarkus source.
  params:
    - name: kustomize-dir
      description: Where should kustomize look for kustomization in source?
    - name: target-namespace
      description: Where to apply the kustomization to
    - name: image-name
      description: Which image to use. Kustomize is taking care of it
  steps:
    - name: build
      image: quay.io/wpernath/kustomize-ubi:latest
      workingDir: $(workspaces.source.path)
      script: |

        cd $(workspaces.source.path)/$(params.kustomize-dir)

        DIGEST=$(cat $(workspaces.source.path)/target/jib-image.digest)

        kustomize edit set image quay.io/wpernath/simple-quarkus:latest=$(params.image-name)@$DIGEST

        kustomize build $(workspaces.source.path)/$(params.kustomize-dir) > $(workspaces.source.path)/target/kustomized.yaml

    - name: apply
      image: 'image-registry.openshift-image-registry.svc:5000/openshift/cli:latest'
      workingDir: $(workspaces.source.path)
      script: |
        oc apply -f $(workspaces.source.path)/target/kustomized.yaml -n $(params.target-namespace)
```

Paste this text into a new file called `kustomize-task.yaml`. As you can see from the contents of the file, this task requires a workspace called `source` and three parameters: `kustomize-dir`, `target-namespace`, and `image-name`. The task contains two steps: `build` and `apply`.

The build step uses the Kustomize image to set the new image and digest. The apply step, finally, uses the OpenShift internal CLI image to apply the Kustomize-created files in the `target-namespace`.

To load the `kustomize-task.yaml` file into your current OpenShift project, simply execute:

```bash
$ oc apply -f kustomize-task.yaml
task.tekton.dev/kustomize configured
```

## Putting it all together
We have now created a pipeline that contains four tasks: `git-clone`, `package`, `build-and-push-image`, and `apply-kustomize`. We have provided the necessary parameters to each task and to the pipeline and we have connected workspaces to it.

Now we have to create the PersistentVolumeClaim (PVC) and a `maven-settings` ConfigMap, which will then be used by the corresponding PipelineRun.

### Creating a maven-settings ConfigMap
If you have a working `maven-settings` file, you can easily reuse it with the maven task. Simply create it via:

```bash
$ oc create cm maven-settings --from-file=/your-maven-settings --dry-run=client -o yaml > maven-settings-cm.yaml
```

If you need to edit the ConfigMap, feel free to do it right now and then execute:

```bash
$ oc apply -f maven-settings-cm.yaml
```

To import the ConfigMap into your current project.

### Creating a PersistentVolumeClaim
Create a new file with the following content and execute `oc apply -f` to import it into your project:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: builder-pvc
spec:
  resources:
    requests:
      storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain

```

This file reserves a PVC with the name `builder-pvc` and a requested storage of 10GB. It’s important to use `persistentVolumeReclaimPolicy: Retain` here, as we want to reuse build artifacts from the previous builds. More on this requirement later in this chapter.

## Running the pipeline
Once you have imported all your artifacts into your current project, you can run the pipeline. To do so, click on the **Pipelines** entry on the left side of the Developer Perspective of OpenShift, choose your created pipeline, and select **Start** from the **Actions** menu on the right side. After you’ve filled in all necessary parameters (Figure 8), you’re able to start the PipelineRun.
![Image 8: Starting the pipeline with all parameters][image-8]

The **Logs** and **Events** cards of the OpenShift Pipeline Editor show, well, all the logs and events. If you prefer to view these things from the command line, use `tkn` to follow the logs of the PipelineRun:

```bash
$ tkn pr
```

The output shows the available actions for PipelineRuns.

```bash
$ tkn pr list
NAME                                       STARTED        DURATION     STATUS
build-and-push-image-run-20211123-091039   1 minute ago   54 seconds   Succeeded
build-and-push-image-run-20211122-200911   13 hours ago   2 minutes   Succeeded
build-and-push-image-ru0vni                13 hours ago   8 minutes    Failed
```

To follow the logs of the last run, simply execute:
```bash
$ tkn pr logs -f -L
```

If you omit the `-L` option, `tkn` lets you choose from the list of PipelineRuns.

You can log, list, cancel, and delete PipelineRuns.

Visual Code also has a Tekton Pipeline extension, which you can also use to edit, build, and execute pipelines.

### Creating a PipelineRun object
In order to start the pipeline via a shell (or from any other application you’re using for CI/CD), you need to create a PipelineRun object, which looks like the following:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: $PIPELINE-run-$(date "+%Y%m%d-%H%M%S")
spec:
  params:
    - name: git-url
      value: https://github.com/wpernath/book-example.git
    - name: git-revision
      value: main
    - name: context-dir
      value: the-source
    - name: image-name
      value: quay.io/wpernath/person-service
    - name: image-username
      value: wpernath
    - name: image-password
      value: *****
    - name: target-namespace
      value: book-tekton
  workspaces:
    - name: shared-workspace
      persistentVolumeClaim:
        claimName: builder-pvc
    - configMap:
        name: maven-settings
      name: maven-settings
  pipelineRef:
    name: build-and-push-image
  serviceAccountName: pipeline
```

Most of the properties of this object are self-explanatory. Just one word on the `serviceAccountName` property: Each PipelineRun runs under a given service account, which means that all pods started along the pipeline are running inside this security context.

OpenShift Pipelines creates a default service account for you, called `pipeline`. If you have Secrets that you want to make available to your PipelineRun, you have to connect them with the service account name. But this requirement is out of topic for this chapter of the book.

The `tekton/pipeline.sh` shell script creates a full version of this PipelineRun based on input parameters.

### Optimizing the pipeline
As earlier output from the logs showed, the first pipeline run takes quite a long time to finish: in my case, approximately 8 minutes. The second still took 2 minutes. I was running the pipelines on a home server, which has modest resources. When you use Tekton on your build farms, times should be much lower because you’re running on dedicated server hardware

But still, the pipelines at this point take way too long.

If you’re looking at the logs, you can see that the `maven` task is taking a long time to finish. This is because Maven is downloading the necessary artifacts again and again on every run. Depending on your internet connection, this takes some time, even if you’re using a local Maven mirror.

On your developer machine, Maven uses the `$HOME/.m2` folder as a cache for the artifacts. The same will be done when you’re running Maven from a task. However, because each PipelineRun runs on a separate set of pods, `$HOME/.m2` is not properly defined, which means the whole cache gets invalidated once the PipelineRun is finished.

Maven allows us to specify `-Dmaven.repo.local` to provide a different path to a local cache. This option is what we can use to solve our problem.

I have created a new Maven task (`maven-caching`), which you can find in the [book's example repository][15]. The file was originally just a copy of the one that came from Tekton Hub. But then I decided to remove the init step, which was building a `maven-settings.xml` file based on some input parameters. Instead, I removed most of the parameters and added a MUST-HAVE `maven-settings` ConfigMap. I believe this makes everything much easier.

As Figure 9 shows, we now have only two parameters: `GOALS` and `CONTEXT_DIR`.

![Image 9: Simplified Maven task][image-9]

The important setting for the `maven` call is shown in the second red box of Figure 9. It simply calls `maven` with the `maven-settings` property and with the parameter indicating where to store the downloaded artifacts.

One note on artifact storage: During my tests of this example, I realized that if the `git-clone` task clones the source to the root directory of the PVC (when no `subdirectory` parameter is given on task execution), the next start of the pipeline will delete everything from the PVC again. And in that case, we once again have no artifact cache.

So you have to provide a `subdirectory` parameter (in my case, I used a global `the-source`) and provide exactly the same value to the `CONTEXT_DIR` parameter to the `maven` calls.

The changes discussed in this section reduce our maven calls dramatically, in my case from 8 minutes to 54 seconds:

```bash
$ tkn pr list
NAME					   STARTED        DURATION     STATUS
build-and-push-image-123   1 minute ago   54 seconds   Succeeded
build-and-push-image-ru0   13 hours ago   8 minutes    Succeeded
```

## Summary of using Tekton pipelines
Tekton is a powerful tool for creating CI/CD pipelines. Because it is based on Kubernetes, it uses extensive concepts from Kubernetes and reduces the maintenance of the tool itself. If you want to quickly start your first pipeline, try to use the OpenShift Developer UI, which you get for free if you’re installing the Operator. This gives you a nice base to start your tests. However, at some point— especially when it comes to optimizations — you need a proper editor to code your pipelines.

One of the biggest advantages of Tekton over CI/CD tools such as Jenkins is that you can reuse all your work for other projects and applications. If you want to standardize the way your pipelines work, build one pipeline and simply specify different sets of parameters for different situations. PipelineRun objects make this possible. The pipeline we have just created in this chapter can easily be reused for all Quarkus-generated applications. Just change the `git-url` and `image-name` parameters. That‘s all. Isn‘t this great?

And even if you’re not satisfied with all the tasks you get from Tekton Hub, use them as bases and build your own iterations out of them, as we did with the optimized Maven task and the Kustomize task in this chapter.

I would not say that Tekton is the easiest technology available to do CI/CD pipelines, but it is definitely one of the most flexible.

However, we have not even talked about [Tekton security][16] and how we are able to provide, for example, Secrets to access your Git repository or the image repository. And we have cheated a little bit about image generation, because we were using the mechanism Quarkus provides. There are other ways of creating images with a dedicated Buildah task.

The next chapter of this book discusses Tekton security, as well as GitOps and Argo CD.

[1]:	https://tekton.dev
[2]:	https://tekton.dev "Tekton Homepage"
[3]:	https://cloud.redhat.com/blog/introducing-openshift-pipelines "Introducing OpenShift Pipelines"
[4]:	https://github.com/code-ready/crc
[5]:	https://github.com/wpernath/book-example/tree/main/person-service "Quarkus Simple"
[6]:	https://github.com/GoogleContainerTools/jib "Google's JIB"
[7]:	https://quay.io/repository/wpernath/quarkus-simple-wow "quay.io"
[8]:	https://www.opensourcerers.org/2021/04/26/automated-application-packaging-and-distribution-with-openshift-part-12/ "Kustomize explained "
[9]:	https://github.com/GoogleContainerTools/jib "Google's Jib"
[10]:	https://github.com/wpernath/book-example
[11]:	https://quarkus.io/guides/container-image#quarkus-container-image-jib_quarkus.jib.base-jvm-image
[12]:	https://hub.tekton.dev
[13]:	https://github.com/wpernath/kustomize-ubi
[14]:	https://quay.io/repository/wpernath/kustomize-ubi
[15]:	https://raw.githubusercontent.com/wpernath/book-example/main/tekton/tasks/maven-task.yaml
[16]:	https://tekton.dev/docs/pipelines/auth/

[image-1]:	Bildschirmfoto%202021-07-02%20um%2012.20.33.png
[image-2]:	Bildschirmfoto%202021-07-02%20um%2012.25.29.png
[image-3]:	quarkus-app-props.png
[image-4]:	Bildschirmfoto%202021-07-08%20um%2007.41.24.png
[image-5]:	Bildschirmfoto%202021-07-13%20um%2015.00.18.png
[image-6]:	DraggedImage.png
[image-7]:	Bildschirmfoto%202021-07-19%20um%2009.54.39.png
[image-8]:	pipeline-run.png
[image-9]:	Bildschirmfoto%202021-07-20%20um%2011.30.29.png