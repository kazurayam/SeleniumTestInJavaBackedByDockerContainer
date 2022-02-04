# Automated UI Test using Selenium backed by Docker Container

## Problem to solve

I am interested in automating Web UI testing in Java/Groovy using Selenium WebDriver. At the same time, I am interested in developing Web Server applications in Python. So I want to execute UI tests written in Java against my Web Application written in Python. Of course, I can do it by the following manual operations.

In a Terminal on Mac, I start a Docker container by

    $ cd ~/tmp
    $ docker run -d -p 80:8080 kazurayam/flaskr-kazurayam:1.1.0

I will open another Terminal, and execute UI tests by

    $ cd $projectDir
    $ gradle test
    ....

When the test has finished, I will go back to the 1st terminal window. I will identify the process id of the docker container:

    $ docker ps --filter publish=80 --filter status=running -q
    fd5ad3b76b13

Once I find the pid, I can stop the process gracefully by:

    $ docker stop fd5ad3b76b13

This procedure is not difficult. But it is cumbersome, easy to make mistakes. I often forget terminating the previous docker container, and try to start another one. Then I get an error saying "the IP port is already in use". Just frustrating.

So, I want to automate starting and stopping any Docker Container on the localhost by my test code in Java.

## Solution

I have developed a code in Java using JUnit 5, which does the following:

-   This test visits and tests a URL "http://127.0.0.1:3080/" using Selenium WebDriver.

-   The URL is served by a process on the localhost, in which a Docker Container runs using a docker image which kazurayam published.

-   The web app was originally developed in Python language by the Pallets project, is published at [flaskr tutorial](https://flask.palletsprojects.com/en/2.0.x/tutorial/)

-   This test automates running and stopping a Docker Container process using commandline commands: `docker run`, `docker ps` and `docker stop`.

-   The [`com.kazurayam.subprocessj.docker.ContainerRunner`](https://github.com/kazurayam/subprocessj/blob/master/src/main/java/com/kazurayam/subprocessj/docker/ContainerRunner.java) class wraps the "docker run" command.

-   The [`com.kazurayam.subprocessj.docker.ContainerFinder`](https://github.com/kazurayam/subprocessj/blob/master/src/main/java/com/kazurayam/subprocessj/docker/ContainerFinder.java) class wraps the "docker ps" command.

-   The [`com.kazurayam.subprocessj.docker.ContainerStopper`](https://github.com/kazurayam/subprocessj/blob/master/src/main/java/com/kazurayam/subprocessj/docker/ContainerStopper.java) class wraps the "docker stop" command.

-   These classes call [`java.lang.ProcessBuilder`](https://www.baeldung.com/java-lang-processbuilder-api) to execute the `docker` command from Java.

-   The [`com.kazurayam.subprocessj.Subprocess`](https://github.com/kazurayam/subprocessj/blob/master/src/main/java/com/kazurayam/subprocessj/Subprocess.java) class wraps the `ProcessBuilder` and provides a simple API for Java application with work with OS commands.

## Description

### Sequence diagram

The following diagram shows the sequence.

![sequence](https://docs/diagrams/out/DockerBackedWebDriverTest_sequence.png)

### Sample code

-   [example.DockerBackedWebDriverTest](https://github.com/kazurayam/subprocessj/blob/master/src/test/java/example/DockerBackedWebDriverTest.java)

<!-- -->

    Unresolved directive in README_.adoc - include::../src/test/java/example/DockerBackedWebDriverTest.java[DockerBackedWebDriverTest]

## How to reuse this

See [build.gradle](https://github.com/kazurayam/subprocessj/blob/master/build.gradle)

## Conclusion

I wanted to perform automated Web UI testings written in Java/Groovy against a web application written in Python. The [Subprocessj 0.3.0](https://mvnrepository.com/artifact/com.kazurayam/subprocessj) library made it possible for me. I am contented with it.

## References for Docker

-   [forum.docker.com, “docker run” cannot be killed with ctrl+c](https://forums.docke.com/t/docker-run-cannot-be-killed-with-ctrl-c/13108/)

-   [docker ps command](https://matsuand.github.io/docs.docker.jp.onthefly/engine/reference/commandline/ps/)

-   [docker run command](https://docs.docker.com/engine/reference/commandline/run/)

-   [docker stop command](https://matsuand.github.io/docs.docker.jp.onthefly/engine/reference/commandline/stop/)

-   [subprocessj project top](https://github.com/kazurayam/subprocessj)

## links

The Subprocessj project’s artifact is available at the Maven Central repository:

-   <https://mvnrepository.com/artifact/com.kazurayam/subprocessj>

The project’s repository is here

-   [the repository](https://github.com/kazurayam/subprocessj/)
