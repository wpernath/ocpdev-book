# Chapter Five: GitOps and Argo CD
The previous chapters discussed the basics of modern application development with Kubernetes. This chapter shows you how to integrate a project into Kubernetes native pipelines to do your CI/CD and automatically deploy your application out of a pipeline run. We discuss the risks and benefits of using GitOps and [Argo CD](https://argoproj.github.io/argo-cd/) in your project and give you some hints on how to use it with Red Hat OpenShift.

## Introduction to GitOps
I can imagine a reader complaining, "We are still struggling to implement DevOps and now you're coming to us with yet another new fancy acronym to help solve all the issues we still have?“ This is something I have heard when I first talked about GitOps during a customer engagement.

The short answer is: DevOps is a cultural change in your enterprise, meaning that Developers and Operations people should talk to each other instead of doing their work secretly behind big walls.

GitOps is an evolutionary way of implementing continuous deployments for the cloud and Kubernetes. The idea behind Gitops is to use the same version control system you're using for your code to store formal descriptions of the infrastructure desired in the test or production environment. These descriptions can be updated as the needs of the environment change, and can be managed through version control just like source code. You automatically gain a history of all the deployments you've done. After each change, an automated process runs (either through a manual step or through automation) to make the production environment match the desired state. The term "healing" is often applied to the process that brings the actual state of the system in sync with the desired state.

**AO: The text you quoted was no longer on the Web site, so I just wrote a description that I thought would be more useful.**

### Motivation Behind GitOps
But why Git? And why now? And what does Kubernetes has to do with all that?

As described earlier in this book, you already should be maintaining a formal description of your infrastructure. Each application you're deploying on Kubernetes has a bunch of YAML files that are required to run your application. Adding those files to your project in a Git repository is just a natural step forward. And if you have a tool that could read those files from the repository and apply them to a specified Kubernetes namespace…wouldn't that be great?

Well, this is GitOps. And Argo CD is one of the available tools to help you do GitOps.

### What Does a Typical GitOps Process Look Like?
One of the most questions people ask most often about GitOps is this: Is it just another way of doing CI/CD? The answer to this question is simply No. GitOps only takes care of the CD part, the delivery part.

Without GitOps, the developer workflow looks like this:
1. A developer implements a change request.
2. Once the developer commits the changes to Git, an integration pipeline is triggered.
3. This pipeline compiles the code, runs all automated tests, and creates and pushes the image.
4. Finally, the pipeline automatically installs the application on the test stage.

With GitOps, the developer workflow looks somewhat different (see Image 1):
1. A developer implements a change request.
2. Once the developer commits the changes to Git, an integration pipeline is triggered.
3. This pipeline compiles the code, runs all automated tests, and creates and pushes the image.
4. The pipeline automatically updates the configuration files' directory in the Git repository to reflect the changes.
5. The CD tool sees a new desired state in Git, which is then synchronized to the Kubernetes environment.

![Image 1: The GitOps delivery model](Bildschirmfoto%202021-07-27%20um%2015.09.27.png)

So you're still using your pipeline based on Tekton, Jenkins, or whatever to do CI. GitOps then takes care of the CD part.

## Argo CD Concepts
Right now (as of version 2.0), the concepts behind Argo CD are quite easy. You register an Argo Application that contains pointers to the necessary Git repository with all the application-specific descriptors such as Deployment, Service, etc., and to the Kubernetes cluster. You might also define an Argo Project, which defines various defaults such as:
- Which source repositories are allowed
- Which destination servers and namespaces can be deployed to
- A whitelist of cluster resources to deploy, such as Deployments, Services, etc.

Once the Application is registered, you can manually start a sync to update the actual environment. Alternatively, Argo CD starts "healing" the application automatically, if the synchronization policy is set to do so.

## The Use Case: Implementing GitOps for our Quarkus-Simple App
We've been using the [quarkus-simple application](https://github.com/wpernath/quarkus-simple "Quarkus Simple") over the course of this book. Let's continue to use it and create a GitOps workflow for it. You can find all the resources discussed here in the `gitops` folder within the `quarkus-simple` application on GitHub.

**AO: The first link below points to Red Hat OpenShift GitOps release notes. I don't think that's the best link to go to about the Operator. The current release notes don't mention the Operator. The page links to a page about Operators, but that page doesn't include GitOps! I suggest you point here to the same URL you use later, https://docs.openshift.com/container-platform/4.8/cicd/gitops/installing-openshift-gitops.html.**

We are going to set up Argo CD (via the [OpenShift GitOps Operator](https://docs.openshift.com/container-platform/4.8/cicd/gitops/gitops-release-notes.html)) on OpenShift 4.8 (via [Red Hat CodeReady Containers](https://github.com/code-ready/crc)). We are going to use [Tekton to build a pipeline](https://www.opensourcerers.org/2021/07/26/automated-application-packaging-and-distribution-with-openshift-tekton-pipelines-part-34-2/), which updates our [quarkus-simple-config](https://github.com/wpernath/quarkus-simple-config) Git repository with the latest image digest of the build. Argo CD should then detect the changes and should start a synchronization of our application.

**NOTE**: Typically, a GitOps pipeline does not directly push changes into the main branch of a configuration repository. Instead, the pipeline should commit into a feature branch or release branch and should create a pull request, so that committers can review changes before they are merged to the main branch.

## The Application Configuration Repository
First of all, let's create a new repository for our application configuration (`quarkus-simple-config`).

One of the main concepts behind GitOps is to represent your the configuration and build parameters of your application as a Git repository. This  repository could either be part of the source code repository or separate. As I am a big fan of [*separation of concerns*](https://deviq.com/principles/separation-of-concerns), we will create a new repository containing [artifacts that we built in earlier chapters using Kustomize](https://www.opensourcerers.org/2021/04/26/automated-application-packaging-and-distribution-with-openshift-part-12/):

```bash
$> tree
└── config
    ├── base
    │   ├── config-map.yaml
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   ├── route.yaml
    │   └── service.yaml
    └── overlays
        ├── dev
        │   └── kustomization.yaml
        ├── prod
        │   └── kustomization.yaml
        └── stage
            ├── apply-health-checks.yaml
            ├── change-env-value.yaml
            └── kustomization.yaml

6 directories, 10 files
```

Of course, there are several ways to structure your config repositories. Some natural choices include:
1. A single configuration repository with all files covering all services and stages
2. A separate configuration repository per service or application, with all files for all stages
3. A separate configuration repositories for each stage of each service

This is completely up to you. But option 1 is probably not optimal because combining all services and stages might make the files hard to read, and does not promote separation of concerns. On the other hand, option 3 may break up information too much, forcing you to maintain hundredths of files for different apps or services. Therefore, option 2 strikes me as a good balance: One repository per app, containing files that cover all stages for that app.

For now, create this configuration repository by copying the files from the  `quarkus-simple/kustomize_ext` directory into the newly created Git repository.

**AO: It's not create when or how the Git repository was created. And it's not clear whether you're talking here about an empty directory or a cloned respository.  Does the reader have to issue a `mkdir` command? Or `git clone`? The latter command does the copying.**

**NOTE**: The original kustomization.yaml file already contains an image section. This should be removed first.

## Installing the OpenShift GitOps Operator
Because the OpenShift GitOps Operator is offered free of charge to OpenShift users and comes quite well preconfigured, I am focusing on its use. If you want to bypass the Operator and dig into Argo CD installation, please feel free to have a look at the official [guides](https://argo-cd.readthedocs.io/en/latest/getting_started/).

The [OpenShift GitOps Operator can easily be installed in OpenShift](https://docs.openshift.com/container-platform/4.8/cicd/gitops/installing-openshift-gitops.html). Just log in as a user with cluster-admin rights and switch to the **Administrator** perspective of the OpenShift console. Then go to the **Operators** menu entry and select **OperatorHub** (Image 2). In the search field, start typing "gitops" and select the GitOps Operator when its panel is shown.

![Image 2: Installing the OpenShift GitOps Operator](Bildschirmfoto%202021-08-04%20um%2009.20.44.png)

Once the Operator is installed, it creates a new newspace called `openshift-gitops` where an instance of Argo CD is installed and ready to be used.

At of time of this writing, Argo CD is not yet configured to use OpenShift authentication, so you have to get the password of the admin user by getting the value of the Secret `openshift-gitops-cluster` in namespace `openshift-gitops`:

```bash
$> oc get secret openshift-gitops-cluster -n openshift-gitops -ojsonpath='{.data.admin\.password}' | base64 -d
```

And this is how to get the URL of your Argo CD instance:

```bash
$> oc get route openshift-gitops-server -ojsonpath='{.spec.host}' -n openshift-gitops
```

## Creating a New Argo Application
The easiest way to create a new Argo Application is by using the GUI provided by Argo CD (Image 3).

![Image 3: Argo CD on OpenShift](Bildschirmfoto%202021-08-04%20um%2009.30.15.png)

 After logging in to the GUI, click **New App** and fill in the required fields shown in Image 4, as follows:

1. Application Name: We'll use `quarkus-simple', the same name as our repository.
2. Project: In our case it's `default`, which was created during Argo CD installation.
3. Syn Policy: Choose wether you want automatic synchronization, which is enabled by the **SELF HEAL** option.
4. Repository URL: Specify your directory with the application metadata (Kubernetes resources).
5. Path: This specifies the subdirectory within the repository that points to the actual files.
6. Cluster URL: Specify your Kubernetes instance. **AO: I'm not sure how to describe it.**
7. Namespace: The OpenShift of Kubernetes namespace to deploy to.

![Image 4: Creating a new Argo CD Application using the GUI](Bildschirmfoto%202021-08-04%20um%2015.53.03.png)

After filling out the fields, click **Create**. All Argo CD objects of the default Argo CD instance will be stored in the namespace `openshift-gitops`, from where you can export them via:

```bash
$> oc get Application/quarkus-simple -o yaml -n openshift-gitops
```

To create an application object in a new Kubernetes instance, open the `quarkus-simple-app.yaml` file exported by the previous command in your preferred editor:

**AO: I would prefer plain text output here instead of the figure**

 ![Image 5: Exported and cleaned-up YAML file of the application object](Bildschirmfoto%202021-08-05%20um%2009.52.29.png)

**AO: What does it mean to remove metadata or clean up the file?**

Remove the metadata from the object file, and then enter the following command to import the application into the predefined Argo CD instance:

```bash
$> oc apply -f quarkus-simple-app.yaml -n openshift-gitops
```

You have to import the application into the `openshift-gitops` namespace. Otherwise, it won't be recognized by Argo CD.

**AO: If the reader didn't choose automatic synchronization, they need to press **Sync**. I'd like some reorganization in the following section. If the reader chose automatic sync, I can understand that they'll immediate trigger the error. But you could also explain why the add-role-to-user command is needed, run it, then tell the reader to press **Sync**. However, you can say that the error will appear if they use automatic sync. Is there a way to avoid the error?**

## First Synchronization
You might notice that the first synchronization takes quite a while and breaks without doing anything except to issue an error message (Image 6).
![Image 6: Argo CD UI synchronization failure](Bildschirmfoto%202021-08-05%20um%2015.26.39.png)

The error arises because the service account of Argo CD does not have the necessary authority to create typical resources in a new namespace. You have to enter the following command for each namespace Argo CD is taking care of:

```bash
$> oc policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n <target namespace>
```

Alternatively, if you prefer to use a YAML description file for this task, create something like the following:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: quarkus-simple-role-binding
  namespace: quarkus-simple
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- kind: ServiceAccount
  name: openshift-gitops-argocd-application-controller
  namespace: openshift-gitops
```

**NOTE**: You could also provide cluster-admin rights to the Argo CD service account. This would have the benefit or allowing Argo CD to everything on its own. The drawback is that Argo is then super user of your Kubernetes cluster.  This might not be very secure.

After you've given the service account the necessary role, you can safely click **Sync** and Argo CD will do the synchronization (Image 7).
![Image 7: Argo CD UI with successful synchronization](Bildschirmfoto%202021-08-05%20um%2015.47.24.png)

If you chose automatic sync during configuration, any change to a file in the application's Git repository will cause Argo CD to check what has changed and start the necessary actions to keep the environment in sync.

## Creating a Tekton Pipeline to Update quarkus-simple-config
We now want to change our pipeline from the previous chapter (Image 8) to be more GitOps'y. But what exactly needs to be done?
![Image 8: Tekton pipeline from Chapter 4](Bildschirmfoto%202021-08-05%20um%2015.52.50.png)

The current pipeline is a development pipeline, which will be used to:
1. Compile and test the code.
2. Create a new image.
3. Push that image to an external registry (in our case [Quay.io](https://quay.io)).
4. Use Kustomize to change the image target.
5. Apply the changes via the OpenShift CLI to a given namespace.

In GitOps, we don't do pipeline-centric deployments anymore. As explained earlier, the final step of our pipeline just updates our `quarkus-simple-config` Git repository with the new version of the image. Instead of the `apply-kustomize` task, we are creating and using the `git-update-deployment` task as final step. This task should clone the config repository, use Kustomize to apply the image changes, and finally push the changes back to GitHub.com.

### A Word On Tekton Security
Because we want to update a private repository, we first need to have a look at [Tekton authentication](https://tekton.dev/docs/pipelines/auth/). Tekton uses specially annotated Secrets with either a *username*/*password* combination or an SSH key. The authentication then produces a `~/.gitconfig` file (or, in case of an image repository, a `~/.docker/config.json` file) and maps it into the step's pod via the run's associated ServiceAccount. That's easy, isn't it? Configuring the process looks like:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-user-pass
  annotations:
    tekton.dev/git-0: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: <cleartext username>
  password: <cleartext password>
```

Once you've filled in `username` and `password`, you can apply the Secret into the namespace where you want to run your newly created pipeline.

```bash
$> oc new-project art-gitops
$> oc apply -f secret.yaml
```

Now you need to either create a new ServiceAccount for your pipeline or update the existing one, which was generated by the OpenShift Pipeline Operator. The pipeline runs completely within the security context of the provided ServiceAccount.

Let's decide on a new ServiceAccount. To see which other Secrets this ServiceAccount requires, execute:

```bash
$> oc get sa/pipeline -o yaml
```
Copy the Secrets to your own ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
	metadata:
  		name: pipeline-bot
secrets:
- name: git-user-pass
```

You don't need to copy the following generated Secrets to your ServiceAccount, because they will be linked automatically with the new ServiceAccount by the Operator:

- `pipeline-dockercfg-`: The default Secret for reading and writing images from and to the internal OpenShift registry.
- `pipeline-token-`: The default Secret for the `pipeline` ServiceAccount. This is used internally.

You also have to create two RoleBindings for the ServiceAccount. Otherwise, you can't reuse the PersistenceVolumes we've been using so far:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: piplinebot-rolebinding1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pipelines-scc-clusterrole
subjects:
  - kind: ServiceAccount
    name: pipeline-bot
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: piplinebot-rolebinding2
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
  - kind: ServiceAccount
    name: pipeline-bot
```

The `edit` Role is mainly used if your pipeline needs to change any Kubernetes metadata in the given namespace. If your pipeline doesn't do things like this, you can safely ignore that Role. In our case, we don't necessary need the edit Role.

### The git-update-deployment Tekton Task
Now that you understand Tekton authentication and have created all the necessary manifests, you are able to focus on the `git-update-deployment` task.

Remember, we want to have a task that does the following:
- Clone the configuration Git repository.
- Update the image digest via Kustomize.
- Commit and push the changes back to the repository.

This means you need to create a task with at least the following parameters:
- `GIT_REPOSITORY`: The configuration repository to clone.
- `CURRENT_IMAGE`: The name of the image in the `deployment.yaml` file.
- `NEW_IMAGE`: The name of the new image to deploy.
- `NEW_DIGEST`: The name of digest of the new image to deploy. This digest is generated in the `build-and-push-image` step that appears both the in the Chapter 4 version and this chapter's version of the pipeline.
- `KUSTOMIZE_PATH`: The path within the `GIT_REPOSITORY` with the `kustomization.yaml` file.

And of course, you need to create a workspace to hold the project files.

Let's have a look at the steps within the task:
```yaml
  steps:
    - name: git-clone
      image: docker.io/alpine/git:v2.26.2
      workingDir: $(workspaces.workspace.path)
      script: |
        rm -rf git-update-digest-workdir
        git clone $(params.GIT_REPOSITORY) git-update-digest-workdir

    - name: update-digest
      image: quay.io/wpernath/kustomize-ubi:latest
      workingDir: $(workspaces.workspace.path)
      script: |
        cd git-update-digest-workdir/$(params.KUSTOMIZATION_PATH)
        kustomize edit set image $(params.CURRENT_IMAGE)=$(params.NEW_IMAGE)@$(params.NEW_DIGEST)

    - name: git-commit
      image: docker.io/alpine/git:v2.26.2
      workingDir: $(workspaces.workspace.path)
      script: |
        cd git-update-digest-workdir

        git config user.email "tekton-bot@my-domain.com"
        git config user.name "My Tekton Bot"

        git add $(params.KUSTOMIZATION_PATH)/kustomization.yaml
        git commit -m "[ci] Image digest updated"

        git push
```

Nothing special here. It's the same things we would do via the CLI. The full task and everything related can be found, as always, in the `gitops/tekton` folder of the [quarkus-simple repository](https://github.com/wpernath/quarkus-simple) on GitHub.

### Creating an extract-digest Tekton Task
The next question is how to get the image digest. Because we are using the [Quarkus image builder](https://www.opensourcerers.org/2021/07/26/automated-application-packaging-and-distribution-with-openshift-tekton-pipelines-part-34-2/) (which in turn is using [Jib](https://cloud.google.com/blog/products/application-development/introducing-jib-build-java-docker-images-better)), we need to create either a step or a separate task that provides content to create a `target/jib-image.digest` file. **AO: I don't see why it's relevant here to mention the Quarkus image builder or Jib. Could you omit the step of getting the digest if you used other tools?**

Because I want to have the `git-update-deployment` task as general-purpose as possible, I have created a separate task that does just this step. The step relies on a Tekton feature known as [emitting results from a task](https://tekton.dev/docs/pipelines/tasks/#emitting-results).

Within the `spec` section of a task, you can define a `results` property. Each result is stored in `$(results.<result-name>.path)`, where the `<result-name>` component is a string that refers to the data in that result. Results are available in all tasks and on the pipeline level through strings in the format:

```bash
$(tasks.<task-name>.results.<result-name>)
```

The following configuration defines the step that extracts the image digest and stores it into a result:

```yaml
spec:
  params:
    - name: image-digest-path
      default: target

  results:
    - name: DIGEST
      description: The image digest of the last quarkus maven build with JIB image creation

  steps:
    - name: extract-digest
      image: quay.io/wpernath/kustomize-ubi:latest
      script: |
		# extract DIGEST
        DIGEST=$(cat $(workspaces.source.path)/$(params.image-digest-path)/jib-image.digest)

		# Store DIGEST into result
        echo -n $DIGEST > $(results.DIGEST.path)
```

### Creating the Pipeline
Now it's time to summarize everything in a new pipeline. Image 9 shows the tasks. The first three are the same as in the Chapter 4 version of this pipeline. We have added the `extract-digest` step as described in the previous section, and end by updating our repository.
![Image 9: The gitops-pipeline](Bildschirmfoto%202021-08-10%20um%2013.34.25.png)

Start by using the [previous non-GitOps pipeline](https://raw.githubusercontent.com/wpernath/quarkus-simple/main/tektondev/pipelines/tekton-pipeline.yaml), which we created in Chapter 4. Remove the last task and add `extract-digest` and `git-update-deployment` as new tasks.

You also need two more parameters on pipeline level:
- config-git-url
- config-dir

You map these parameters to the `git-update-deployment` task, as shown in Image 10.
![Image 10: Parameter mapping](Bildschirmfoto%202021-08-10%20um%2013.42.31.png)

### Testing the Pipeline
To start the pipeline, navigate within the Developer perspective of the OpenShift UI in your `art-gitops` project, select the pipeline, and click **Start** in the **Actions** menu. Fill in the necessary parameters and click **Start** button at the bottom right (Image 11).

![Image 11: Starting the pipeline](Bildschirmfoto%202021-08-11%20um%2008.37.56.png)

Have a look at Chapter 4 for more on starting and testing pipelines.

For your convenience, I have created a Bash script, called `gitops/tekton/pipeline.sh` which can be used to initialize your namespace:

```bash
$> oc new-project art-gitops
$> cd gitops/tekton/
$> ./pipeline.sh init -u <git-user-name> -p <git-hash>
$> ./pipeline.sh start -u <quay-user-name> -p <quay-hash>
```

Whenever the pipeline is successfully executed, you should see an updated message in the `quarkus-simple-config` Git repository. And you should see that Argo CD has initiated a synchronization process, which ends with a redeployment of the quarkus application.

## Creating a stage-release Pipeline
What does a staging pipeline look like? We need a process which does the following in our case (Image 12):
1. Clone the config repository.
2. Create a release branch (e.g., `release-1.2.3`).
3. Get the image digest. (In our case, we extract the image out of the current development environment.)
4. Tag the image in the image repository (e.g., `quay.up/wpernath/quarkus-simple-wow:1.2.3`).
5. Update the configuration repository and point the stage configuration to the newly tagged image.
6. Commit and push the code back to the Git repository.
7. Create a pull or merge request.

![Image 12: The staging pipeline](Bildschirmfoto%202021-08-20%20um%2010.27.07.png)

These tasjs are followed by a manual process where a test specialist accepts the pull request and merges the content from the branch back into the main branch. Then Argo CD takes the changes and updates the running staging instance in Kubernetes.

You can use the Bash script I created to start the staging pipeline, creating release 1.2.5, as follows:

```bash
$> ./pipeline.sh stage -r 1.2.5
```

Image 13 shows a repository in the GitHub UI after a push.

![Image 13: The Git release after executing the pipeline](Bildschirmfoto%202021-08-20%20um%2010.48.51.png)

### Setup of the Pipeline
The `git-clone` and `git-branch` steps use existing ClusterTasks, so there is nothing to explain here except one new Tekton feature: [Conditional execution of a task](https://tekton.dev/docs/pipelines/pipelines/#guard-task-execution-using-when-expressions) by using a "When" expression.

In our case, the `git-branch` task should be executed only when a `release-name` is specified. The corresponding YAML code in the pipeline looks like:

```yaml
      when:
        - input: $(params.release-name)
          operator: notin
          values:
            - ""
```

The new `extract-digest` task is using `yq` to extract the digest out of the `kustomization.yaml` file. The command looks like:

```bash
$> yq eval '.images[0].digest' $(workspaces.source.path)/$(params.kustomize-dir)/kustomization.yaml
```

The result of this call is stored in the task's `results` field.

### The tag-image Task
The `tag-image` task uses the `skopeo-copy` ClusterTask, which requires a source image and a target image. The original use case of this task was to copy images from one repository to another (for example, from the local repository up to an external Quay.io repository). However, you can also use this task to tag an image in a repository. The corresponding parameters for the task are:

```yaml
    - name: tag-image
      params:
        - name: srcImageURL
          value: >-
            docker://$(params.target-image)@$(tasks.extract-digest.results.DIGEST)
        - name: destImageURL
          value: >-
            docker://$(params.target-image):$(params.release-name)
        - name: srcTLSverify
          value: 'false'
        - name: destTLSverify
          value: 'false'
      runAfter:
        - extract-digest
      taskRef:
        kind: ClusterTask
        name: skopeo-copy
[...]
```

`skopeo` uses an existing docker configuration if it finds one in the home directory of the current user. This means for us that we have to create another Secret with the following content:
```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    tekton.dev/docker-0: https://quay.io
  name: quay-push-secret
type: kubernetes.io/basic-auth
stringData:
  username: <use your quay.io user>
  password: <use your quay.io token>
```

Also update the ServiceAccount accordingly:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-bot
secrets:
- name: git-user-pass
- name: quay-push-secret
```

After you've applied the configuration changes, the `skopeo` task is able to authenticate to Quay.io and do its work.

### Creating the release
The final `create-release` task does more or less exactly the same work as the task we've already used in the `gitops-dev-pipeline` task:
- Run Kustomize to set the image in the `config/overlays/stage/kustomization.yaml` file.
- Commit and push the changes back to GitHub.

## Challenges
Of course, you still face quite some challenges if you're completely switching over to do GitOps. Some challenges spring from the way Kubernetes intrinsically works, which has pros and cons by itself. Other challenges exist because originally, Git was meant to be used by people who are able to analyze merge conflicts and apply them manually.

However, nobody says GitOps would be the only and *de facto* method of doing CD nowadays. If GitOps is not right for your environment, simply don't use it.

But let's discuss some of the challenges and possible solutions.

### Order-dependent Deployments
Although order-dependent deployments are not necessarily a best practice, the reality is that nearly every large application does have dependencies, and these must be fulfilled before the installation process can continue. Two examples:

- Before my app can start, I must have a properly installed and configured database.
- Before I can install an instance of a Kafka service, an Operator must be installed on Kubernetes.

Fortunately, Argo CD has a solution for those scenarios. Like with Helm Charts, you're able to define [sync phases and so called *waves*](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/).

Argo CD defines three sync phases:
- **PreSync**: Before the synchronization starts
- **Sync**: The actual synchronization phase
- **PostSync**: After the synchronization is complete

Within each phase, you can define waves to list activities to perform. However, presync and postsync can contain only *hooks*. A hook could be of any Kubernetes type, such as a pod or job, but could also be of type TaskRun or PipelineRun (if the corresponding CRDs are already installed in your Cluster).

Waves can be defined by annotating your Kubernetes resources with the following annotation:

```yaml
metadata:
  annotations:
	argocd.argoproj.io/sync-wave: <+/- number>
```

Argo CD sorts all resources first by the phase, then by the wave, and finally by type and name. If you know that some resources need to be applied before others, simply group them via the annotation. By default, Argo CD uses wave zero for any resources and hooks.

### Nondeclarative Deployments
Nondeclarative deployments are simply lists of steps; they are also called *imperative*. For most of us, they the most familiar type of deployment. For instance, if you're creating a hand-over document for the OPS guys, you are providing imperative instructions about how to install the application. And most of us are used to creating installation scripts for more complex applications.

However, the preferred way of installing applications with Kubernetes is to through declarative deployments. These specify the service, the deployment, persistent volume claims, secrets, config maps etc.

If this declaration is not enough, you have to provide a script to configure a special resource—for example, updating the structure of a database or doing a backup of the data first.

As mentioned earlier, Argo CD manages synchronization via phases and [executes resource hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/) when necessary. So you can define a presync hook to execute a database schema migration or fill the database with test data. You might create a postsync hook to do some tiding or health checks.

Let's create such a hook:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: post-sync-run
  name: my-final-run
  annotations:
    argocd.argoproj.io/hook: PostSync
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: post-install-job
          image: "registry.access.redhat.com/ubi8/ubi-minimal:latest"
          command:
          - /bin/sh
          - -c
          - |
            echo "WELCOME TO the post installation hook for Argo CD "
            echo "-------------------------------------------------"
            echo "Here we could start integration tests or health checks"
            echo "..."

            sleep 10
```

This hook just runs a simple job that prints a message and then waits for 10 seconds. The only difference between this sync and a "normal" job is the `argocd.argoproj.io/hook` annotation.

As a result of a successful synchronization, you could also start a Tekton PipelineRun or any Kubernetes resource that is already registered in your cluster.

**NOTE**: Please make sure to add your sync jobs to the base `kustomization.yaml` file. Otherwise, they won't be processed.

Image 14 shows the results of a sync.

![Image 14: The synchronization log of Argo CD with a presync and a postsync hook](Bildschirmfoto%202021-08-13%20um%2000.19.58.png)

### Git Repository Management
In a typical enterprise with at least dozen of applications, you could easily end up with a lot of Git repositories. This complexity can make it hard to manage them properly, especially in terms of security (controlling who is able to push what).

If you want to use one single configuration Git repository for all your applications and stages, please keep in mind that Git was never meant to automatically resolve merge conflicts. Instead, conflict resolution sometimes needs to be done manually. Be careful in such a case and plan your releases thoroughly. Otherwise, you might end up in not being able to automatically create and merge release branches anymore.

**AO: I don't see a need for the following figure.**
 ![Image 15: Possible merge conflicts  with automatic concurrent pull requests](Unbenanntes_Projekt.jpg)

### Managing Secrets
Another important task requiring a lot of thought is proper Secret management. A Secret contains access tokens to mission-critical external applications, such as databases, SAP systems, or in our case GitHub and Quay.io). You don't want to make that confidential informations publicly accessible in a Git repository.

Instead, think about a solution like [Hashicorp Vault](https://www.hashicorp.com/products/vault/secrets-management), where you are able to centrally manage your Secrets.

## Summary
The desire to automatically deploy the latest code into production is as old as information technology, I suppose. Automation *is* important, as it makes everybody in the release chain more productive and helps to create reproducible and well-documented deployments.

There are many ideas and even more tools out there for how to implement automation. The easiest way, of course, is to create scripts for deployment that do exactly what you need. The downside of this is that scripts might become unmanageable after some time. *AO: You also tend to do a lot of copying, and then if you want to make a change you need to find and correct each script.**

With Kubernetes and GitOps, you can define everything as a file, which helps you store everything in a Version control system and use the conveniences it provides.

Kubernetes bases its infrastructure on YAML files, which makes it easy to reuse what you already know as a developer or administrator: just use Git and store the complete infrastructure description in a repository. You work through the repository to create releases, roll the latest release back if there was a bug in it, keep track of any changes, and create an automated process around deployments.

Other benefits of using Git include:
- Every commit has a description.
- Every commit could be required to be bound to an issue previously submitted (i.e., no commit without providing an issue number).
- Everything is auditable.
- Pull or merge requests.

You get all those features for free, which helps you become more productive.

A tool like Argo CD, which keeps the repository and the Kubernetes cluster in sync, is just the natural evolution of DevOps automation.

## Final note
This book started as a set of articles in January 2021. I originally wanted just some notes to help me in my day-to-day work. Then I realized that there are tons of things out there that need to be explained. People in my classes were telling me that they are overwhelmed with all the news around Kubernetes and OpenShift, so I decided to not only talk about it, but also write and blog about it.

The positive feedback I got from readers around the world motivated me a lot to continue writing and to add this chapter. If I'd export all the articles as PDFs, I would get more than 110 pages in DIN A4 format. I wasn't aware that I was able to write so much on a development topic.

Thanks a lot for reading, and for all your feedback.
