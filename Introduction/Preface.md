# Preface
During my day-to-day job, I am frequently explaining and demonstrating, to interested developers and architects, the benefits of using OpenShift and modern Kubernetes platforms for their own projects. I started writing some [blog posts][1]about it just because I wanted to record what I was telling the attendees of my classes.

Then I realized that there are tons of things out there that need explanation. People in my classes are telling me that they’re overwhelmed with all the news around Kubernetes and OpenShift and don’t really understand all the technologies, or the benefits of using those technologies in their own projects. They wish to have a practical guide taking them through the whole story. So I decided to not only talk about it, but also write a book about it.

As there are a lot of books available to explain the theoretical justification for the benefits of GitOps and Kubernetes, I decided to start from the other side: I have a use case and want to go through it from the beginning to the end.

## What to expect
You might already know GitOps. You might understand the benefits of using Kubernetes from a developer’s perspective, because it allows you to set up deployments using declarative configurations. You might also understand container images or Tekton Pipelines or Argo CD.

But did you already use everything together in one project? Did you already make your hands dirty with Tekton? With Argo CD?

If you want to delve all the way down in GitOps, you might benefit from somebody giving you some hints here and there.

This is why the book was written. Do GitOps from scratch.

## The use case
Typically, most modern software projects need to design and implement one or more services. This service could be a RESTful microservice or a reactive front end. To provide an example that is meaningful to a large group of readers, I have decided to write a RESTful microservice, called `person-service`, which requires a third-party software stack (in my case, a PostgreSQL database).

This service has API methods to read data from the database and to update, create, and delete data using JSON. Thus, it’s a simple CRUD service.

## Chapter overview
This book tries to proceed as you would when developing your own services. The initial question is always (after understanding the business requirements, of course), what software stack should be used. In my case I have decided to use the Java language with the [Quarkus][2] framework.

Chapter 1 explains why I’m using Quarkus and how to develop your microservice with it.

Once you’ve developed your code, you need to understand how to move it to your target platform. This also requires some understanding of the target platform. This what Chapter 2 is all about: Understanding container images and all the Kubernetes manifest files, and how to easily modify them for later use.

After you’ve decided on your target platform, you might also want to decide how to distribute your application. This task includes application packaging, and choosing the right format to package it. In Chapter 3 we discuss possible solutions, with examples and more detail.

So now that you understand how to package and distribute your application, let’s set up a process to automate the tasks of building the sources and deploying your service to your test and production stages. Chapter 4 explains how to use Tekton to set up an integration pipeline for your service.

And finally, Chapter 5 sets up GitOps for your service.

## Summary
This book aims to be a blueprint or a guide for your journey to GitOps with OpenShift and Kubernetes.

Thank you for reading the book.


[1]:	https://www.opensourcerers.org/2021/04/26/automated-application-packaging-and-distribution-with-openshift-basic-development-principles-part-14/
[2]:	https://quarkus.io
