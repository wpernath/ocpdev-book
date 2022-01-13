# Preface
During my day to day job I am explaining and demonstrating interested developers and architects the benefits of using OpenShift and modern Kubernetes platforms for their own projects. I started writing some [blog posts][1]about it just because I wanted to write down, what I am telling the attendees of my classes. 

Then I realized that there are tons of things out there which need to be explained. People in my classes were telling me that they are overwhelmed with all the news around Kubernetes and OpenShift and that they don’t really understand all the technologies and the benefits of using those in their own projects and that they wish to have a practical guide through the whole story, so I decided to not only talk about it, but also write a book about it.  

As there are a lot of books available which are explaining the theoretical thesis about the benefits of GitOps and Kubernetes, I decided to start from the other side: I have a use case and want to go through it from the beginning to the end. 

## What to expect
You might already know GitOps, you might understand the benefits of using Kubernetes from a developer’s perspective as it allows us to use declarative deployments. You might also understand container images or Tekton Pipelines or ArgoCD. 

But did you already use everything together in one project? Did you already made your hands dirty on Tekton? On ArgoCD?

So if you want to go all the way down to GitOps, you might need somebody who’s giving you some hints here and there. 

This is why the book was written. Do GitOps from scratch. 

## The use case 
Typically, most modern software projects have one or more services which they need to design and implement. This service could be a RESTful micro service or a reactive frontend or something like this. I have decided to write a RESTful micro service, called `person-service`, which requires a third party software stack (e.g. a PostgreSQL database). 

This service has API methods to read data from the database and to update, create and delete data using JSON. It’s a simple CRUD service. 

## Chapter overview
This book is written like you would start developing your own services. The initial question is always (after understanding the business requirements, of course), what software stack should be used. In my case I have decided to use Java as language and [Quarkus][2] as framework. 

The first chapter explains, why I have been using Quarkus and how to develop your micro service with it.  

Once you’ve developed your code, you need to understand how to move it to your target platform. This also includes understanding the target platform. This is the second chapter all about: Understanding container images and all the Kubernetes manifest files and how to easily modify them for later use.

After you’ve decided for your target platform, you might also want to understand how you’re able to distribute your application. This includes application packaging as well as choosing the right format to package it. In chapter three we are going to discuss all those possible solutions with examples in more detail. 

So now you’re understanding how to package and distribute your application, let’s start to setup a process to automate building the sources and deploying your service to your stages. Chapter four explains how to use Tekton to setup an integration pipeline for your service. 

And finally we are going to discuss GitOps for your service. 

## Summary
This book aims to be a blueprint or a guide for your journey to GitOps with OpenShift and Kubernetes. 

Thank you for reading this book. 
 

[1]:	https://www.opensourcerers.org/2021/04/26/automated-application-packaging-and-distribution-with-openshift-basic-development-principles-part-14/
[2]:	https://quarkus.io