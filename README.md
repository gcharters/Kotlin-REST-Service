# Kotlin REST Service

A simple "hello" service written in Kotlin, along with an integration test, also written in Kotlin.  The build is maven-based and assumes code is only being written in Kotlin.  If you want to mix Java with Kotlin you need to follow the instructions here: https://kotlinlang.org/docs/reference/using-maven.html .

## Build
Build the service and run the test using:

```
mvn clean install
```
The build creates a war containing the application and the Kotlin pre-reqs.  The build also creates a runnable jar using Open Liberty.

## Start the service using maven

```
mvn liberty:run-server
```
## Start the service using the runnable jar

```
cd target
java -jar kotlinHello.jar
```

## Use the service

```
http://localhost:9080/kotlinHello/hello/Graham
```
