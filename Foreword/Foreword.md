# Foreword
The job of a software developer has never been as approachable as it is nowadays. There are plenty of free resources provided by great communities to start from, allowing you to achieve quick successes with only little effort. But only a few of us remained in the role of a pure programmer, and young professionals face different challenges from those that we had one to two decades ago. The DevOps mindset and company restructurings made us all "software engineers."

The name "software engineer" is not just a rebranding of these creepy, nocturnal keyboard maniacs; it mainly means two things: Building software is established as an engineering discipline (one that needs to be learned), and it's not only about producing source code anymore, but also about creating and operating complex systems composed by the right components for every purpose.

That's where two of the most exciting technologies of our time come in. I'm not talking about virtual backgrounds in video conferences and the unmute button. Of course: Quarkus and Kubernetes are meant. Quarkus satisfies a developers needs as well as Kubernetes's demands and allows for actual software engineering high in the clouds.

Although Quarkus prepares Java ecosystem inhabitants well for the cloud, there's some more of the "operations part" to consider (remember: engineering, not only developing). In regards of operations, another "thing" has became quite popular—at least as a keyword; but considered closely, is that the most natural operating model for Kubernetes: [GitOps][1].

There were (and still are) many discussions and sophisticated concepts about how to solve continuous builds and deployments (CI/CD). Yes, that's a very important topic, but honestly—in the world of Kubernetes—it's a piece of cake. Building, testing, and integrating software modules is straightforward, especially when targeting to containers. And continuous delivery? That can be really complicated, I admit—that's why we actually don't do it by ourselves.

We've known "Infrastructure as Code" for some time; it has become common practice in the most cloud environments. With GitOps we do exactly the same regarding application deployment and configuration. The desired state of Kubernetes resources is described declaratively and versioned in Git. Operators running in the target environment are continuously validating and reconciling this state. The process of continuous delivery is fully automated by specialized components such as the projects from the [Cloud Native Computing Foundation (CNCF)][2], Argo CD and Flux CD, and of course the build-ins of OpenShift.

How all of this nicely plays together, peppered with lots of examples, can be found in this book. At the time my team adopted Kubernetes, GitOps wasn't widely known yet. Sticking to the principles of GitOps fostered the transition toward independent engineering teams and increased the overall quality of our solutions. Since then, we have helped a lot of teams succeed with the same journey. This job becomes much easier now that we can simply hand out the piece of literature you're looking at. So take Wanja's red pill and follow him down the rabbit hole of modern software delivery.

*Florian Heubeck, Principal Engineer at MediaMarktSaturn Technology & Java User Group Ingolstadt Leader*

[1]:	https://opengitops.dev/
[2]:	https://www.cncf.io/