# Chapter 1: Creating the sample service
This books needs a good example to demonstrate the power of working with Kubernetes and OpenShift. As an application that will be familiar and useful to most readers, we’ll create a REST-based microservice in Java that reads data from and writes data to a database.

I have to say, I like [Quarkus][1]. I have been coding for over two decades with Java and JEE. I have been using most of the frameworks out there (does anyone remember Struts or JBoss Seam or SilverStream?). I’ve even created code generators to make my life easier with EJBs (1.x and 2.x). But all of those frameworks and ideas had one thing in common: they tried to minimize the efforts for the developers but they had other drawbacks.

And then back in 2020 when I thought, there is nothing out there which could really positively surprise me, I had a look at Quarkus. This is my personal story about Quarkus.

So this chapter is all about creating a microservice with Quarkus. If you want to understand more about Quarkus, feel free to get one of the other books available on the [Red Hat developers page][2]. **(AO: Currently, there seems to be only one book on the topic, which I happened also to edit: https://developers.redhat.com/e-books/quarkus-spring-developers. So maybe you should just point to it.)**

## First steps
Quarkus has a [Get Started][3]page. Go there to have a look at how to install the command line tool, which is called `quarkus`.

After you’ve installed the `quarkus` tool, create a new project by executing:

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
    │   ├── docker
    │   │   ├── Dockerfile.jvm
    │   │   ├── Dockerfile.legacy-jar
    │   │   ├── Dockerfile.native
    │   │   └── Dockerfile.native-distroless
    │   ├── java
    │   │   └── org
    │   │       └── wanja
    │   │           └── demo
    │   │               └── GreetingResource.java
    │   └── resources
    │       ├── META-INF
    │       │   └── resources
    │       │       └── index.html
    │       └── application.properties
    └── test
        └── java
            └── org
                └── wanja
                    └── demo
                        ├── GreetingResourceTest.java
                        └── NativeGreetingResourceIT.java

15 directories, 13 files
```

If you want to test what you have done so far, call:
```bash
$> mvn quarkus:dev
```

Or if you prefer to use the Quarkus CLI tool, you can also call:
```bash
$> quarkus dev
```

This compiles all sources and starts the development mode of your project, where you don’t need to specify any runtime environment (Tomcat, JBoss, etc.).

Let’s have a look at the generated `GreetingResource.java` file, which you can find under `src/main/java/org/wanja/demo`:

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

So it seems that, if `quarkus:dev` is running, you should have an endpoint reachable at `localhost:8080/hello` in a browser on that system. Let’s have a look. For testing of REST endpoints, you can use either `curl` or the much newer client called [httpie][4]. I prefer to the newer one:

```bash
$> http :8080/hello
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
content-length: 14

Hello RESTEasy
```

This was easy. But still nothing really new. Let’s go a little bit deeper.

Let’s change the string `Hello RESTEasy` into something new and call the service again (but without restarting `quarkus dev`—that’s a key point to make).

```bash
$> http :8080/hello
HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
content-length: 7

Hi Yay!
```

OK, this is getting interesting now. You don’t have to recompile or restart Quarkus to see your changes in action. Let’s have a look at the [Quarkus documentation about configuring your application][5], because we don’t want to use hard-coded strings.

To reconfigure the application, open  `src/main/resources/application.properties` in your preferred editor and create a new property. For example:

```bash
app.greeting=Hello, dear quarkus developer!
```

And then go into the GreetingResource and create a new property on the class level:

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

Again, you haven’t recompiled or restarted the services. Quarkus is watching for any changes in the source tree and takes the required actions automatically. **(AO: Because we talk about "healing" later in Chapter 5, it might be worth introducing that term here. But it also might confuse the reader.)**

This is already great. Really. But let’s move on.

## Creating a database client
The use case for this book should not be a simple hello service. We want to have a database client that reads from writes and to a database. After [reading the corresponding documentation][6], I decided to use Panache here, as it seems to reduce the work I have to do dramatically.

First you need to add the required extensions to your project. The following command installs a JDBC driver for PostgreSQL and everything to be used for ORM:

```bash
$> quarkus ext add quarkus-hibernate-orm-panache quarkus-jdbc-postgresql
Looking for the newly published extensions in registry.quarkus.io
[SUCCESS] ✅  Extension io.quarkus:quarkus-hibernate-orm-panache has been installed
[SUCCESS] ✅  Extension io.quarkus:quarkus-jdbc-postgresql has been installed
```

### Java code for database operations
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

Well, according to the docs, this should be the Person entity, which maps directly to a `person` table in our PostgreSQL database. All public properties will be mapped automatically to the corresponding entity in the database. If you don’t want that, you need to specify the `@Transient` annotation. That can’t be that easy, can it? **(AO: I don’t understand the rhetoric in the previous sentence. Are you being sarcastic? Will you return to this issue?)**

You also need a `PersonResource` class to act as a REST endpoint. Let’s create such a simple class:

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

Right now, this class has exactly one method, `getAll()`, which simply returns a list of all persons sorted by the `last_name` column.

### Enabling the database
Now, we need to tell Quarkus that we want to use a database. And then we need to find a way to start a PostgreSQL database locally. But one step at a time.

Open the `application.properties` and add some properties in there:

```java
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.log.format-sql=true
quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.sql-load-script=import.sql

quarkus.datasource.db-kind=postgresql
```

And then let’s make a simple SQL import script to fill some basic data into the database. Create a new file called `src/main/resources/import.sql` and put the following lines in there: **(AO: "Ms" is more popular now than "Mrs", which looks pretty old-fashioned)**

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

You can now restart `quarkus dev` with everything you need:

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

The first time I started Quarkus, I expected exceptions because there was no PostgreSQL database installed locally on my laptop. But… no… No exception upon startup. How could that be?

## Quarkus Dev Services
Every developer has faced situations where they either wanted to quickly test some new feature or had to quickly fix a bug in an existing application. The workflow is mostly the same:
- Setting up the local IDE
- Cloning the source code repository
- Checking dependencies for databases or other infrastructure software components
- Installing the dependencies locally (a Redis server, an Infinispan server, a database, ApacheMQ, or whatever is needed)
- Making sure everything is properly set up
- Creating and implementing the bug fix or the feature

In short, it takes quite some time to actually start implementing what you have to implement.

This is where Quarkus Dev Services come into play. As soon as Quarkus detects that there is a dependency on a third-party component (database, MQ, cache, …) and you have Docker Desktop installed on your developer machine, Quarkus starts the component for you. You don’t have to configure anything. It just happens.

Have a look at the [official Quarkus documentation][7] to see which components are currently supported in this manner in `dev` mode.

## Testing the database client
So you don’t have to install and configure a PostgreSQL database server locally on your laptop. This is great. Let’s test your service now to prove that it works.

```bash
$> http :8080/person
HTTP/1.1 500 Internal Server Error
Content-Type: text/html;charset=UTF-8
content-length: 113

Could not find MessageBodyWriter for response object of type: java.util.ArrayList of media type: application/json
```

OK. Well. It does not work. You need a `MessageBodyWriter` for this response type. If you have a look at the class `PersonResource`, you can see that we are directly returning a response of type `java.util.List<Person>`. And we have a global producer annotation of `application/json`. We need a component that translates the result into a JSON string.

This can be done through the `quarkus-resteasy-jsonb` or `quarkus-resteasy-jacksonb` extension. We are going to use the first one by executing:

```bash
$> quarkus ext add quarkus-resteasy-jsonb
[SUCCESS] ✅  Extension io.quarkus:quarkus-resteasy-jsonb has been installed
```

If you now call the endpoint again, you should see the correctly resolved and formatted output.

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
For a well rounded create-read-update-delete (CRUD) service, you still have to implement methods to add, delete and update a person from the list. Let’s do it now.

### Creating a new person
The code snippet to create a new person is quite easy. Just implement another method, annotate it with `@POST`  and `@Transactional`, and that’s it.

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

The only relevant method we call in this method is `persist()`, called on a given `Person` instance. This is known as the [active record pattern][8] and is described in the official documentation. **(AO: Consider pointing here to https://quarkus.io/guides/hibernate-orm-panache#solution-1-using-the-active-record-pattern instead.)**

Let’s have a look to see whether it works:

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

The returned JSON indicates that we did what we intended.

### Updating an existing person
The same is true for updating a person. Use the `@PUT` annotation and make sure you are providing a path parameter, which you have to annotate with `@PathParam`:

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

Then test it:

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
And finally, let’s create a `delete` method, which works in the same way as the `update()` method:

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

And let’s check whether it works:
```bash
$> http DELETE :8080/person/1
HTTP/1.1 204 No Content
```

This is a correct response with a code in the 200 range.

## Preparing for CI/CD
Until now, everything you did was mainly for local development purposes. With just a few lines of code, you’ve been able to create a complete database client. You did not even have to worry about setting up a local database for testing.

But how can you specify real database properties when entering test or production stages?

Quarkus supports [configuration profiles][9]. Properties marked with a given profile name are used only if the application runs in that particular profile. By default, Quarkus supports the following profiles:
dev::
Gets activated when you run your app via `quarkus dev`
test::
Gets activated when you are running tests
prod::
The default profile if the app is not started in the `dev`profile

In our case, we want to specify database-specific properties only in `prod` mode. If we try to specify a database URL in `dev` mode, for example, Quarkus would not start the corresponding dev services. **(AO: The previous sentence needs context. Are you saying that you can’t specify the properties because Quarkus can’t apply them in dev mode? Or are you saying that you choose not to specify a database in dev mode, so Quarkus doesn’t apply the properties?)**

Our configuration therefore is:

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

Quarkus also supports the use of [property expressions][10]. For instance, if your application is running on Kubernetes, you might want to specify the  datasource username and password via a Secret. In this case, use the `${PROP_NAME}` expression format to refer to the property that was set in the file. Those expressions are evaluated when they are read. The property names are either specified in the `application.properties` file or can come from environment variables.

Now your application is [prepared for CI/CD][11] and for production.

## Moving the app to OpenShift
Quarkus provides extensions to generate manifest files for Kubernetes or [OpenShift][12]. Let’s add the extensions to our `pom.xml` file:

```bash
$> quarkus ext add jib openshift
```

The `jib` extension will help you generate a container image out of the application. The `openshift` extension will generate the necessary manifest files to deploy the application on—well—OpenShift.

Let’s specify the properties accordingly:

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

Now build the application container image via:
```bash
$> mvn package -Dquarkus.container-image.push=true
```

This command also pushes the image to [Quay.io][13] as `quay.io/wpernath/singer:v1.0.0`. Quarkus is using [Jib][14] to build the image.

After the image is built, you can install the application into OpenShift by applying the manifest file:

```bash
$> oc apply -f target/kubernetes/openshift.yml
service/person-service configured
imagestream.image.openshift.io/person-service configured
deployment.apps/person-service configured
route.route.openshift.io/person-service configured
```

Then create a PostgreSQL database instance in the same namespace from the corresponding template. You can install the database from the OpenShift console by clicing on **+Add→Developer Catalog→Database→PostgreSQL** and filling in meaningful properties for the service name, user name, password and database name. You could alternatively execute the following command from the shell to instantiate a PostgreSQL server in the current namespace:

```bash
$> oc new-app postgresql-persistent \
	-p POSTGRESQL_USER=wanja \
	-p POSTGRESQL_PASSWORD=wanja \
	-p POSTGRESQL_DATABASE=wanjadb \
	-p DATABASE_SERVICE_NAME=wanjaserver
```

Suppose you’ve specified the database properties in `application.properties` like this:

```java
%prod.quarkus.datasource.username=${DB_USER:wanja}
%prod.quarkus.datasource.password=${DB_PASSWORD:wanja}
%prod.quarkus.datasource.jdbc.url=jdbc:postgresql://${DB_HOST:wanjaserver}/${DB_DATABASE:wanjadb}
```

Quarkus takes the values after the colon as defaults, which means you don’t have to create those environment values in the `Deployment` file for this test. But if you want to use a Secret or ConfigMap, have a look at the [corresponding extension][15] for Quarkus.

After restarting the `person-service` you should see that the database is used and that the `person` table was created. But there is no data in the database, because you’ve defined the corresponding property to be used in dev mode only.

So fill the database now:
```bash
$> http POST http://person-service-paul.apps.art8.ocp.lan/person firstName=Jimi lastName=Hendrix salutation=Mr

$> http POST http://person-service-paul.apps.art8.ocp.lan/person firstName=Joe lastName=Cocker salutation=Mr

$> http POST http://person-service-paul.apps.art8.ocp.lan/person firstName=Carlos lastName=Santana salutation=Mr

```

You should now have three singers in the database. To verify, call:
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
Do you want to create a native executable out of your Quarkus app? That’s easily done by calling:

```bash
$> mvn package -Pnative -DskipTests
```

However, this call would require you to set up [GraalVM][16] locally. GraalVM is a code generation tool that creates native C code from Java (JVM) input. If you don’t want to install and set up GraalVM locally or if you’re always building for a container runtime, you could instruct Quarkus to [do a container build][17] as follows:

```bash
$> mvn package -Pnative -DskipTests -Dquarkus.native.container-build=true
```

If you also define `quarkus.container-image.build=true`, Quarkus will produce a native container image, which you could then use deploy to a Kubernetes cluster.

Try it. And if you’re using OpenShift 4.9, you could have a look at the `Observe` register within the Developer Console. This page tells me, for instance, that my OpenShift 4.9 instance is installed on an Intel NUC with a Core i7 with 6 cores.

Using a native image instead of a JVM one changes quite a few things:
- Startup time decreases from 1.2sec (non-native) to 0.03sec (native)
- Memory usage decreases from 120MB (non-native) to 25MB (native)
- CPU utilization drops to 0.2% of the requested CPU time

## Summary
Using Quarkus dramatically reduces the lines of code you have to write. As you have seen, creating a simple REST CRUD service is a piece of cake. If you then want to move your app to Kubernetes, it’s just a matter of adding another extension to the build process.

Thanks to the Dev Services, you’re even able to do fast prototyping without worrying about install many third-party applications, such as databases.

Minimizing the amount of boilerplate code makes your application easier to maintain, and lets you focus on what you really have to do: implementing the business case.

This is why I fell in love with Quarkus.

Now let’s have a deeper look into working with images on Kubernetes and OpenShift.

[1]:	https://quarkus.io
[2]:	https://developers.redhat.com/e-books
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
[16]:	https://www.graalvm.org
[17]:	https://quarkus.io/guides/building-native-image#container-runtime