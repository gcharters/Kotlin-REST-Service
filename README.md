# How to Run Kotlin on Liberty, and Why You'd Want to Do It

Kotlin is a fast-growing alternative language to Java that runs on a JVM, which is being used for front-end development on Android as well as on servers. As a JVM language, you can use all of the tools and processes you know and love, but with a new language. In fact, you can mix Kotlin and Java code in the same application, meaning you can also continue to use your favorite frameworks and libraries. This lab will go from the basics of Kotlin code and how it compares to Java, to building and deploying REST APIs using Open Liberty.

## Introduction

Kotlin is not a new language, but there's no doubt it's received a huge boost with the announcement last year that it was now <a href="https://developer.android.com/kotlin/index.html">an official language for Android</a> development.  As Kotlin use grows on Android, so does the need for Kotlin on the server.  "Backend for frontend" (BFF) is the concept of having server-side components that are aligned with the front-end user experience.  Both are usually created and maintained by the same team and so using the same technologies makes a lot of sense.  This, and the fact that Kotlin is simply a really good, modern language means demand for server-side Kotlin is growing.

### Approaches

This QuickLab shows a simple approach to using Kotlin on the Liberty application server.  It focuses on enabling existing Liberty users to adopt Kotlin by enabling the use of all of Liberty's existing Enterprise Java APIs from Kotlin.  It does this by showing how to include the Kotlin runtime, and how to develop, build and test a Kotlin-based application.

## Kotlin and Enterprise Java

New languages often struggle to gain adoption early on because they're missing the richness of frameworks and standards that communities provide over time.  Kotlin has cleverly avoided this problem by being fully interoperable with Java.  This means that, for example, to create a RESTful backend in Kotlin, you can make use of existing Java server-side technologies, such as those supported by Eclipse MicroProfile (e.g. JAX-RS, JSON-P, etc).

In this Quick Lab you'll clone a Kotlin RESTful service sample that includes an example Kotlin test and maven build configuration.  You'll see how the service is implemented, tested and built and then build and try the service out.

### Getting started

Clone the Kotlin sample repository:

`git clone https://github.com/gcharters/Kotlin-REST-Service.git`

There are four main parts to the sample:

1. Maven build - `pom.xml`
2. Server configuration - `src/main/liberty/config/server.xml`
3. REST Service - `src/main/kotlin/gcc/kt/rest/HelloService.kt`
4. Test case - `src/test/kotlin/gcc/kt/rest/it/HelloServiceIT.kt`

### The build

Take a look at the `pom.xml`.  The build is maven-based and makes use of the plugins you would expect when developing a RESTful web service for Liberty.  It uses the <a href="https://github.com/WASdev/ci.maven">liberty-maven-plugin</a> to work with the Open Liberty server - create a server package, start/stop the server, and so on. It uses the <a href="https://maven.apache.org/plugins/maven-war-plugin/">maven-war-plugin</a> to build a WAR file containing the Kotlin REST application, and the <a href="http://maven.apache.org/surefire/maven-failsafe-plugin/">maven-failsafe-plugin</a> for running integration tests. The plugin you wouldn't usually see is the <a href="https://kotlinlang.org/docs/reference/using-maven.html">kotlin-maven-plugin</a> to compile the Kotlin code. 

Note, the maven build in this sample only compiles Kotlin code; if you want to compile Kotlin and Java in the same project you need to use the <a href="https://kotlinlang.org/docs/reference/using-maven.html#compiling-kotlin-and-java-sources">Compiling Kotlin and Java sources</a> configuration.

All but one of the dependencies in the maven pom are the ones you would use when writing a JAX-RS Java REST service. The one that's new is for the Kotlin language library:

```XML
<dependency>  
    <groupId>org.jetbrains.kotlin</groupId> 
    <artifactId>kotlin-stdlib</artifactId>
    <version>${kotlin.version}</version>  
</dependency>  
```

The absence of an explicit maven `<scope/>` for this dependency means it will be packaged in the WAR file, as it's required at runtime. There are other approaches we could take, such as adding it as a <a href="https://www.ibm.com/support/knowledgecenter/en/SSD28V_9.0.0/com.ibm.websphere.wlp.core.doc/ae/cwlp_sharedlibrary.html">shared or global library</a> or a <a href="https://www.ibm.com/support/knowledgecenter/en/SSD28V_9.0.0/com.ibm.websphere.wlp.core.doc/ae/twlp_feat_develop.html">Liberty feature</a>, but this is the simplest. The downside is every app you build this way will have its own copy of Kotlin. It's not a large library, but the more copies you have, the more you have to patch if there are any issues (e.g. security vulnerabilities, etc).

### The server configuration

The server configuration is identical to that of a Java RESTful service, just specifying a dependency on the JAX-RS feature, the ports to be used and the deployment of the application.  Note, the name of the application will be used as the `context root` of the service:

```XML
<server description="Sample Kotlin REST server">
    <featureManager>
        <feature>jaxrs-2.1</feature>
    </featureManager>

    <httpEndpoint httpPort="9080" httpsPort="9443" id="defaultHttpEndpoint" />

    <webApplication id="kotlinHello" location="kotlinHello.war" name="kotlinHello" />
</server>
```

### The service

The RESTful service implementation will generally look familiar to anyone who has used JAX-RS.  The annotations are exactly the same, and essentially the differences are syntactic, albeit a more concise syntax.  In an more advanced service implementation it would be possible to take advantage of more Kotlin features. 

```Java
@Path("/hello")
@ApplicationPath("/")
@Produces("application/json")
class HelloService : Application() {
    
    @GET
    @Path("/{name}")
    @Produces(MediaType.APPLICATION_JSON)
    fun sayHello(@PathParam("name") name: String): Greeting {

        println("HelloService sayHello called: " + name)
        return Greeting("Hello", name)
    }
}
```

The Greeting object used by the service is a little more interesting as this is able to make use of <a href="https://kotlinlang.org/docs/reference/data-classes.html">Kotlin's data classes</a>.  If you're familiar with the effort required to write a Java bean with two properties, getters and setters, then you'll appreciate the conciseness of Kotlin's data classes.

```Java
data class Greeting(val message: String, val name: String)
```


### The test case

The test case is very basic, just checking that it gets an HTTP `200` OK return code. It uses JUnit, but as you can see, it's written in Kotlin. This is again exploiting the full Java interoperability and means you don't have to mix Java and Kotlin in your project. 

The location of the service to test is built up from values provided by the <a href="http://maven.apache.org/surefire/maven-failsafe-plugin/">maven-failsafe-plugin</a> as system properties . The request itself is made using the JAX-RS Client (introduced in JAX-RS 2.0).

```Java
class HelloServiceIT {

    @Test
    fun testApplication() {

        // Set up the path to the service
        val port = System.getProperty("liberty.test.port")
        val contextName = System.getProperty("app.context.root")
        val path = "hello"
        val person = "Fred"
        val url = "http://localhost:" + port + "/" + contextName 
                                      + "/" + path + "/" + person

        // Make the request
        val client = ClientBuilder.newClient()
        val target = client.target(url)
        val response = target.request().accept(MediaType.APPLICATION_JSON).get()

        // Test we got an OK response
        assertEquals("Incorrect response code from " + url, 200, response.getStatus())

        response.close()
    }
}
```

### Try it out

Build the project: `mvn install`

You should see that the build succeeds and the Kotlin integration test ran:

```
Running gcc.kt.rest.it.HelloServiceIT
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.997 sec - in gcc.kt.rest.it.HelloServiceIT

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
```

Start the server: `mvn liberty:run-server`

You should see that the application is ready:

```
[INFO] [AUDIT   ] CWWKT0016I: Web application available (default_host): http://localhost:9080/kotlinHello/
[INFO] [AUDIT   ] CWWKZ0001I: Application kotlinHello started in 0.373 seconds.
[INFO] [AUDIT   ] CWWKF0012I: The server installed the following features: [servlet-3.1, json-1.0, jaxrs-2.0, jaxrsClient-2.0].
[INFO] [AUDIT   ] CWWKF0011I: The server kotlinHelloServer is ready to run a smarter planet.
```

Hit the REST service endpoint from a browser: `http://localhost:9080/kotlinHello/hello/Fred`

You should see the response: 

```Json
{

    "name": "Fred",
    "message": "Hello"

}
```

### Modify the application

You can now make modifications to the application.  In an editor, edit the Greeting data class: `src/main/kotlin/gcc/kt/rest/Greeting.kt`

Add a new import for the Date class and modify the data class to add the current date.  The result should look as follows:

```Java
package gcc.kt.rest

import java.util.Date

data class Greeting(val message: String, val name: String, val date: String = Date().toString())
```

Recompile the service: `mvn compile`

If the server is running it will automatically pick up the change.  If it isn't, start the server again: `mvn liberty:run-server`

Hit the REST service endpoint to see the change: `http://localhost:9080/kotlinHello/hello/Fred`

You should see a result like this:

```Json
{
    "message": "Hello",
    "name": "Fred",
    "date": "Wed Mar 21 13:56:44 PDT 2018"
}
```

### Summary

You've seen how easy it is to start using Kotlin to write server-side applications by exploiting all the existing Enterprise Java capabilities in Liberty.  You've also seen, in the use of a Kotlin Data Class, just one example of how Kotlin can make it simpler to write application code.


