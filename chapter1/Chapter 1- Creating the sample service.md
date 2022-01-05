# Chapter 1: Creating the sample service
I have to say, I like Quarkus. I have been coding for over two decades with Java and JEE. I have been using most of the frameworks out there (does anyone remember Struts or JBoss Seam or SilverStream?). I’ve even created code generators to make my life easier with EJBs (1.x and 2.x). But all of those frameworks and ideas had one thing in common: they tried to minimize the efforts for the developers but they had other drawbacks.

And then back in 2020 when I thought, there is nothing out there which could really positively surprise me, I had a look at Quarkus. This is my personal story about Quarkus. 

## The use case
For the other chapters in this book, we need a good example to demonstrate the power of working with Kubernetes and OpenShift. So let’s create a REST based micro service which reads and writes data from/into a database. 

This chapter is all about creating the micro service with Quarkus. If you want to understand more about Quarkus, feel free to get one of the other books available on the [Red Hat developers page][1]. 

## First steps
[Quarkus][2] has a [Get Started ][3]page. Go there to have a look how to install the command line tool called `quarkus`. 

After you’ve installed the quarkus tool, simply create a new project by executing:

```bash
$> quarkus create app org.wanja.demo:person-service:1.0.0
Looking for the newly published extensions in registry.quarkus.io
-----------

applying codestarts...
📚  java
🔨  maven
📦  quarkus
📝  config-properties
🔧  dockerfiles
🔧  maven-wrapper
🚀  resteasy-codestart

-----------
[SUCCESS] ✅  quarkus project has been successfully generated in:
--> /Users/wpernath/Devel/quarkus/person-service
-----------
Navigate into this directory and get started: quarkus dev
```

This will create an initial maven project with the following structure:

```bash
$> tree
.
├── README.md
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src
    ├── main
    │   ├── docker
    │   │   ├── Dockerfile.jvm
    │   │   ├── Dockerfile.legacy-jar
    │   │   ├── Dockerfile.native
    │   │   └── Dockerfile.native-distroless
    │   ├── java
    │   │   └── org
    │   │       └── wanja
    │   │           └── demo
    │   │               └── GreetingResource.java
    │   └── resources
    │       ├── META-INF
    │       │   └── resources
    │       │       └── index.html
    │       └── application.properties
    └── test
        └── java
            └── org
                └── wanja
                    └── demo
                        ├── GreetingResourceTest.java
                        └── NativeGreetingResourceIT.java

15 directories, 13 files
```

If you want to test what you have done so far, simply call
```bash
$> mvn quarkus:dev
```

Or if you prefer to use the Quarkus CLI tool, you can also call
```bash
$> quarkus dev
```

This compiles all sources and starts the development mode of your project, where you don’t need to specify any runtime environment (Tomcat, JBoss…). 

Let’s have a look at the generated `GreetingResource.java` which you can find under `src/main/java/org/wanja/demo`:

```java
package org.wanja.demo;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/hello")
public class GreetingResource {

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "Hello RESTEasy";
    }
}
```

So it seems, if `quarkus:dev` is running, we should have an endpoint under `localhost:8080/hello`. Let’s have a look. For testing of REST endpoints, you can either use `curl` or the much newer client called [httpie][4]. I prefer to use the newer one. 

```bash
$> http :8080/hello
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
content-length: 14

Hello RESTEasy
```

This was easy. But still nothing really new. Let’s get a little bit deeper. 

Let’s change the string `Hello RESTEasy` into something new and call the service again (of course without restarting `quarkus dev`).

```bash
$> http :8080/hello
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
content-length: 7

Hi Yay!
```

OK, so this is getting interesting now. You don’t have to recompile or restart Quarkus to see your changes in action. Let’s have a look at [configuring our application][5], because we don’t want to use hard-coded strings. 

For this open  `src/main/resources/application.properties` in your preferred editor and create a new property. For example:

```bash
app.greeting=Hello, dear quarkus developer!
```

And then go into the GreetingResource and create a new property on class level:

```java
package org.wanja.demo;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import org.eclipse.microprofile.config.inject.ConfigProperty;

@Path("/hello")
public class GreetingResource {

    @ConfigProperty(name="app.greeting")
    String greeting;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return greeting;
    }
}
```

Test your changes by calling the REST endpoint again:

```bash
$ http :8080/hello
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
content-length: 25

Hello, quarkus developer!
```

Again, we haven’t recompiled or restarted the services. Quarkus is watching for any changes in the source tree and takes the required actions automatically. 

This is already great. Really. But let’s move on.

## Creating a database client
The use case should not be about a simple hello service. We want to have a database client, which reads and writes from / into a database. After [reading the corresponding docs][6], I decided to use Panache here, as it seems to reduce the work I have to do dramatically.

First you need to add the required extensions to your project. 

```bash
$> quarkus ext add quarkus-hibernate-orm-panache quarkus-jdbc-postgresql
Looking for the newly published extensions in registry.quarkus.io
[SUCCESS] ✅  Extension io.quarkus:quarkus-hibernate-orm-panache has been installed
[SUCCESS] ✅  Extension io.quarkus:quarkus-jdbc-postgresql has been installed
```

This call installs a JDBC driver for PostgreSQL and everything to be used for ORM. 

The next step is to create an entity. Let’s call that entity `Person`, so you’re going to create a `Person.java` file. 

```java
package org.wanja.demo;

import javax.persistence.Column;
import javax.persistence.Entity;

import io.quarkus.hibernate.orm.panache.PanacheEntity;

@Entity
public class Person extends PanacheEntity {
	@Column(name="first_name")
    public String firstName;

	@Column(name="last_name")
    public String lastName;

    public String salutation;
}
```

Well, according to the docs, this should be the Person entity, which maps directly to a table `person` in our PostgreSQL database. All public properties will be mapped automatically to the corresponding entity in the database. If you don’t want that, you need to specify the `@Transient` annotation. That can’t be that easy, can it?

You also need a `PersonResource` class which acts as REST endpoint. Let’s create such a simple class:

```java
package org.wanja.demo;

import java.util.List;

import javax.ws.rs.Consumes;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import io.quarkus.panache.common.Sort;

@Path("/person")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public class PersonResource {
    
    @GET
    public List<Person> getAll() throws Exception {
        return Person.findAll(Sort.ascending("last_name")).list();
    }
}
```

Right now, this class has exactly one method `getAll()` which simply returns a list of all persons, sorted by the column `last_name`.

Now, we need to tell Quarkus that we really want to use a database. And then we need to find a way to start a PostgreSQL database locally. But one after another. 

Open the `application.properties` and add some properties in there:

```java
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.log.format-sql=true
quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.sql-load-script=import.sql

quarkus.datasource.db-kind=postgresql
```

And then let’s have a simple SQL import script to fill some basic data into the database. Create a new file called `src/main/resources/import.sql` and put the following lines in there. 

```java
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Doro', 'Pesch', 'Mrs');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Bobby', 'Brown', 'Mr');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Curt', 'Cobain', 'Mr');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Nina', 'Hagen', 'Mrs');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Jimmi', 'Henrix', 'Mr');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Janis', 'Joplin', 'Mrs');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Joe', 'Cocker', 'Mr');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Alice', 'Cooper', 'Mr');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Bruce', 'Springsteen', 'Mr');
insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Eric', 'Clapton', 'Mr');
```

If you are now restarting `quarkus dev`, you should at least have everything you need. 

```bash
$> quarkus dev
2021-12-15 13:39:47,725 INFO  [io.qua.dat.dep.dev.DevServicesDatasourceProcessor] (build-26) Dev Services for the default datasource (postgresql) started.
Hibernate:

    drop table if exists Person cascade
__  ____  __  _____   ___  __ ____  ______
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/
2021-12-15 13:39:48,869 WARN  [org.hib.eng.jdb.spi.SqlExceptionHelper] (JPA Startup Thread: <default>) SQL Warning Code: 0, SQLState: 00000

2021-12-15 13:39:48,870 WARN  [org.hib.eng.jdb.spi.SqlExceptionHelper] (JPA Startup Thread: <default>) table "person" does not exist, skipping
Hibernate:

    drop sequence if exists hibernate_sequence
2021-12-15 13:39:48,872 WARN  [org.hib.eng.jdb.spi.SqlExceptionHelper] (JPA Startup Thread: <default>) SQL Warning Code: 0, SQLState: 00000
2021-12-15 13:39:48,872 WARN  [org.hib.eng.jdb.spi.SqlExceptionHelper] (JPA Startup Thread: <default>) sequence "hibernate_sequence" does not exist, skipping
Hibernate: create sequence hibernate_sequence start 1 increment 1
Hibernate:

    create table Person (
       id int8 not null,
        first_name varchar(255),
        last_name varchar(255),
        salutation varchar(255),
        primary key (id)
    )

Hibernate:
    insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Doro', 'Pesch', 'Mrs')
Hibernate:
    insert into person(id, first_name, last_name, salutation) values (nextval('hibernate_sequence'), 'Bobby', 'Brown', 'Mr')
```

The first time I started it, I expected exceptions because there was no PostgreSQL database installed locally on my laptop. But… no… No exception upon startup. How could that be?

## Quarkus Dev Services
Every developer has faced situations where they either wanted to quickly test some new feature or had to quickly bug fix an existing application. The workflow is mostly the same:
- Setting up the local IDE
- Cloning the source code repository
- Checking dependencies for databases or other infrastructure software components
- Installing the dependencies locally (a Redis, an infinispan server, a database, ApacheMQ…)
- Making sure, everything is properly setup 
- Create and implement the bug fix or the feature

It takes quite some time to actually start implementing what you have to implement.

This is where Quarkus Dev Services come into play. As soon as Quarkus detects that there is a dependency to a 3rd party component (database, MQ, cache…) and you have Docker Desktop installed on your developer machine, Quarkus starts the component for you. You don’t have to configure anything. It just happens. 

Have a look at the [official Quarkus documentation][7] to see which components are currently supported by this way.

## Testing the database client
So you don’t have to install and configure a PostgreSQL database server locally on your laptop. This is great. Let’s test your service now to prove that it works.

```bash
$> http :8080/person
HTTP/1.1 500 Internal Server Error
Content-Type: text/html;charset=UTF-8
content-length: 113

Could not find MessageBodyWriter for response object of type: java.util.ArrayList of media type: application/json
```

OK. Well. It does not work. You need a `MessageBodyWriter` for this response type. If you have a look at the class `PersonResource`, you can see that we are directly returning a response of type `java.util.List<Person>`. And we have a global producer annotation of `application/json`. We need a component which translates the result into a JSON string. 

This can be done by the extension `quarkus-resteasy-jsonb` or `quarkus-resteasy-jacksonb`. We are going to use the first one by executing:

```bash
$> quarkus ext add quarkus-resteasy-jsonb
[SUCCESS] ✅  Extension io.quarkus:quarkus-resteasy-jsonb has been installed
```

If you’re now calling the above endpoint again, you should see the correctly resolved and formatted output.

```bash
$> http :8080/person
HTTP/1.1 200 OK
Content-Type: application/json
content-length: 741

[
    {
        "firstName": "Bobby",
        "id": 2,
        "lastName": "Brown",
        "salutation": "Mr"
    },
    {
        "firstName": "Eric",
        "id": 11,
        "lastName": "Clapton",
        "salutation": "Mr"
    },
    {
        "firstName": "Curt",
        "id": 4,
        "lastName": "Cobain",
        "salutation": "Mr"
    },
...
```

## Finalizing the CRUD REST service
For a real CRUD service, you still have to implement methods to add, delete and update a person from the list. Let’s do it now.

### Create a new person
The code snippet to create a new person is quite easy. Just implement another method, annotate it with `@POST`  and `@Transactional` and that’s it.

```java
    @POST
    @Transactional
    public Response create(Person p) {
        if (p == null || p.id != null)
            throw new WebApplicationException("id != null");
        p.persist();
        return Response.ok(p).status(200).build();
    }
```

So the only relevant call we do in this method is calling `persist()` on a given `Person` instance. This is called the [Active Record Pattern][8] and is described in the official docs. 

Let’s have a look if it works.

```bash
$> http POST :8080/person firstName=Carlos lastName=Santana salutation=Mr
HTTP/1.1 200 OK
Content-Type: application/json
content-length: 69

{
    "firstName": "Carlos",
    "id": 12,
    "lastName": "Santana",
    "salutation": "Mr"
}
```

### Updating an existing person
The same is true for updating a person. Use the `@PUT` annotation and make sure you are providing a path parameter, which you have to annotate with `@PathParam`. 

```java
    @PUT
    @Transactional
    @Path("{id}")
    public Person update(@PathParam Long id, Person p) {
        Person entity = Person.findById(id);
        if (entity == null) {
            throw new WebApplicationException("Person with id of " + id + " does not exist.", 404);
        }
        if(p.salutation != null ) entity.salutation = p.salutation;
        if(p.firstName != null )  entity.firstName = p.firstName;
        if(p.lastName != null)    entity.lastName = p.lastName;
        return entity;
    }
```

Let’s test it:

```bash
$> http PUT :8080/person/6 firstName=Jimi lastName=Hendrix
HTTP/1.1 200 OK
Content-Type: application/json
content-length: 66

{
    "firstName": "Jimi",
    "id": 6,
    "lastName": "Hendrix",
    "salutation": "Mr"
}
```

### Deleting an existing person
And finally, let’s create a `delete` method, which works exactly like the `update()` method:

```java
    @DELETE
    @Path("{id}")
    @Transactional
    public Response delete(@PathParam Long id) {
        Person entity = Person.findById(id);
        if (entity == null) {
            throw new WebApplicationException("Person with id of " + id + " does not exist.", 404);
        }
        entity.delete();
        return Response.status(204).build();
    }
```

And let’s check if it works:
```bash
$> http DELETE :8080/person/1 
HTTP/1.1 204 No Content
```

## Preparing for CI/CD
Until now everything you did was mainly for local development purposes. With just a few lines of code, you’ve been able to create a complete database client. You even did not have to worry about setting up a local database for testing. 

But how could we specify real database properties if we are going into TEST or PROD stages? 

Quarkus supports [configuration profiles][9]. Properties marked with a given profile name will only be used if the application runs in that particular profile. By default, Quarkus supports 3 profiles:
- dev - Gets activated when you run your app via `quarkus dev`
- test - This gets activated when you are running tests
- prod - This is the default profile if the app is not started in `dev`profile

In our case, we want to specify database specific properties only in `prod` mode, because if we are specifying for example a database URL in `dev`, Quarkus would not start the corresponding dev services.

```java
# only when we are developing
%dev.quarkus.hibernate-orm.database.generation=drop-and-create
%dev.quarkus.hibernate-orm.sql-load-script=import.sql

# only in production
%prod.quarkus.hibernate-orm.database.generation=update
%prod.quarkus.hibernate-orm.sql-load-script=no-file

# Datasource settings... 
# note, we only set those props in prod mode
quarkus.datasource.db-kind=postgresql
%prod.quarkus.datasource.username=${DB_USER}
%prod.quarkus.datasource.password=${DB_PASSWORD}
%prod.quarkus.datasource.jdbc.url=jdbc:postgresql://${DB_HOST}/${DB_DATABASE}
```

Quarkus also supports the usage of [Property Expressions][10]. For instance, if your application is running on Kubernetes, you might want to specify the  datasource username and password via a Secret. In this case, simply use the format `${PROP_NAME}`. Those expressions will be evaluated when they are read. The property names are either specified within the `application.properties` file or could be coming from environment variables.

Now your application is [prepared for CI/CD][11] and also for going into production. 

## Let’s move the app to OpenShift
Quarkus provides extensions to [generate Kubernetes or OpenShift][12] specific manifest files. Let’s add them to our `pom.xml` file:

```bash
$> quarkus ext add jib openshift
```

The `jib` extension will help you to generate a container image out of the application. The `openshift` extension on the other hand will generate the necessary manifest files to deploy the application on - well - OpenShift. 

And let’s specify the properties accordingly

```java
# Packaging the app
quarkus.container-image.builder=jib
quarkus.container-image.image=quay.io/wpernath/singer:v1.0.0
quarkus.openshift.route.expose=true
quarkus.openshift.deployment-kind=Deployment

# resource limits
quarkus.openshift.resources.requests.memory=128Mi
quarkus.openshift.resources.requests.cpu=250m
quarkus.openshift.resources.limits.memory=256Mi
quarkus.openshift.resources.limits.cpu=500m

```

Now let’s build the application container image via:
```bash
$> mvn package -Dquarkus.container-image.push=true 
```

This will also push the image to [quay.io][13] as `quay.io/wpernath/singer:v1.0.0`. Quarkus is using [JIB][14] to build the image.

After the image was build, you can install the application into OpenShift by applying the manifest file:

```bash
$> oc apply -f target/kubernetes/openshift.yml
service/person-service configured
imagestream.image.openshift.io/person-service configured
deployment.apps/person-service configured
route.route.openshift.io/person-service configured
```

Then create a PostgreSQL database instance in the same namespace from the corresponding template (click on `+Add -> Developer Catalog -> Database --> PostgreSQL` and provide meaningful properties for service name, user name, password and database name). You could also execute the following command on the shell to instantiate a PostgreSQL server in the current namespace:

```bash
$> oc new-app postgresql-persistent \
	-p POSTGRESQL_USER=wanja \
	-p POSTGRESQL_PASSWORD=wanja \
	-p POSTGRESQL_DATABASE=wanjadb \
	-p DATABASE_SERVICE_NAME=wanjaserver
```

If you’re specifying the database properties in `application.properties` like this:

```java
%prod.quarkus.datasource.username=${DB_USER:wanja}
%prod.quarkus.datasource.password=${DB_PASSWORD:wanja}
%prod.quarkus.datasource.jdbc.url=jdbc:postgresql://${DB_HOST:wanjaserver}/${DB_DATABASE:wanjadb}
```

Quarkus is using the values after the colon as defaults which means you don’t have to create those environment values in the `Deployment` for this test. But if you want to use a Secret or ConfigMap, have a look at the [corresponding extension][15] for Quarkus. 

After restarting the `person-service` you should see that the database will be used and that the person table was created. But there is no data in the database as we’ve defined the corresponding property to be used in dev mode only. 

So let’s fill the database now:
```bash
$> http POST http://person-service-paul.apps.art8.ocp.lan/person firstName=Jimi lastName=Hendrix salutation=Mr

$> http POST http://person-service-paul.apps.art8.ocp.lan/person firstName=Joe lastName=Cocker salutation=Mr

$> http POST http://person-service-paul.apps.art8.ocp.lan/person firstName=Carlos lastName=Santana salutation=Mr

```

You should now have 3 singers in the database. To verify, call:
```bash
$> http http://person-service-paul.apps.art8.ocp.lan/person
HTTP/1.1 200 OK

[
    {
        "firstName": "Joe",
        "id": 2,
        "lastName": "Cocker",
        "salutation": "Mr"
    },
    {
        "firstName": "Jimi",
        "id": 1,
        "lastName": "Hendrix",
        "salutation": "Mr"
    },
    {
        "firstName": "Carlos",
        "id": 3,
        "lastName": "Santana",
        "salutation": "Mr"
    }
]

```

## Becoming native
Do you want to create a native executable out of your quarkus app? That’s easily possible by simply calling:

```bash
$> mvn package -Pnative -DskipTests
```

However, this call would require you to setup GraalVM locally. If you don’t want to install and setup GraalVM locally or if you’re always building for a Container runtime, you could easily instruct Quarkus to [do a container build][16].

```bash
$> mvn package -Pnative -DskipTests -Dquarkus.native.container-build=true
```

If you’re also providing the flag `quarkus.container-image.build=true`, Quarkus will produce a native container image, which you could then use easily on a Kubernetes cluster.

Try it. And if you’re using OpenShift 4.9, you could have a look at the `Observe` register within the Developer Console. My OpenShift 4.9 is installed on an Intel NUC with a Core i7 with 6 cores. 

Using a native image instead of a JVM one has changed quite a few things:
- Startup time from 1.2sec (non-native) to 0.03sec (native)
- Memory usage from 120MB (non-native) to 25MB (native)
- CPU utilization dropped to 0.2% of the requested CPU time

## Summary
Using Quarkus dramatically reduces the lines of code you have to write. As you have seen, creating a simple REST CRUD service is just a piece of cake. If you then want to move your app to Kubernetes, it’s just a matter of adding another extension to the build process.

Thanks to the Dev Services, you’re even able to do fast prototyping without worrying to install 3rd party apps like databases. 

Minimizing the amount of boilerplate code makes your app easier to maintain - and lets you focus on what you really have to do: implementing the business case. 

This is why I fell in love with Quarkus. 

Now let’s have a deeper look into working with Images on Kubernetes and OpenShift. 

[1]:	https://developers.redhat.com/e-books
[2]:	https://quarkus.io
[3]:	https://quarkus.io/get-started/
[4]:	https://httpie.io
[5]:	https://quarkus.io/guides/config
[6]:	https://quarkus.io/guides/hibernate-orm-panache
[7]:	https://quarkus.io/guides/dev-services
[8]:	https://quarkus.io/guides/hibernate-orm-panache#most-useful-operations
[9]:	https://quarkus.io/guides/config-reference#profiles
[10]:	https://quarkus.io/guides/config-reference#property-expressions
[11]:	https://www.opensourcerers.org/2021/07/26/automated-application-packaging-and-distribution-with-openshift-tekton-pipelines-part-34-2/
[12]:	https://quarkus.io/guides/deploying-to-openshift
[13]:	https://quay.io
[14]:	https://github.com/GoogleContainerTools/jib
[15]:	https://quarkus.io/guides/kubernetes-config
[16]:	https://quarkus.io/guides/building-native-image#container-runtime