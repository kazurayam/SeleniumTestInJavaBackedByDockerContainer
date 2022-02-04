<!-- START doctoc -->
<!-- END doctoc -->

# Selenium Test in Java backed by Docker Container

@author kazurayam
@date 4 Feb, 2022

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

![sequence](https://kazurayam.github.io/SeleniumTestInJavaBackedByDockerContainer/diagrams/out/sequence.png)

### Sample code

-   [example.DockerBackedWebDriverTest](https://github.com/kazurayam/SeleniumTestInJavaBackedByDockerContainer/blob/master/src/test/java/example/DockerBackedWebDriverTest.java)

<!-- -->

    package example;

    import com.kazurayam.subprocessj.docker.ContainerFinder;
    import com.kazurayam.subprocessj.docker.ContainerFinder.ContainerFindingResult;
    import com.kazurayam.subprocessj.docker.ContainerRunner;
    import com.kazurayam.subprocessj.docker.ContainerRunner.ContainerRunningResult;
    import com.kazurayam.subprocessj.docker.ContainerStopper;
    import com.kazurayam.subprocessj.docker.ContainerStopper.ContainerStoppingResult;
    import com.kazurayam.subprocessj.docker.model.ContainerId;
    import com.kazurayam.subprocessj.docker.model.DockerImage;
    import com.kazurayam.subprocessj.docker.model.PublishedPort;
    import io.github.bonigarcia.wdm.WebDriverManager;
    import org.junit.jupiter.api.AfterAll;
    import org.junit.jupiter.api.AfterEach;
    import org.junit.jupiter.api.BeforeAll;
    import org.junit.jupiter.api.BeforeEach;
    import org.junit.jupiter.api.Test;
    import org.openqa.selenium.By;
    import org.openqa.selenium.WebDriver;
    import org.openqa.selenium.WebElement;
    import org.openqa.selenium.chrome.ChromeDriver;

    import java.io.File;
    import java.io.IOException;
    import java.nio.file.Files;

    import static org.junit.jupiter.api.Assertions.assertEquals;
    import static org.junit.jupiter.api.Assertions.assertNotNull;

    /**
     * A JUnit5 test case.
     * This test visits and tests a URL "http://127.0.0.1:3080/" using Selenium WebDriver.
     * The URL is served by a process in which a Docker Container runs
     * using a docker image which kazurayam published.
     * The web app was originally developed in Python language by the Pallets project,
     * is published at
     * - https://flask.palletsprojects.com/en/2.0.x/tutorial/
     *
     * This test automates running and stopping a Docker Container process
     * using commandline commands: `docker run` and `docker stop`.
     *
     * The `com.kazurayam.subprocessj.docker.ContainerRunner` class wraps the "docker run" command.
     * The `com.kazurayam.subprocessj.docker.ContainerFinder` class wraps the "docker ps" command.
     * The `com.kazurayam.subprocessj.docker.ContainerStopper` class wraps the "docker stop" command.
     *
     * These classes call `java.lang.ProcessBuilder` to execute the `docker` command from Java.
     * The `com.kazurayam.subprocessj.Subprocess` class wraps the `ProcessBuilder` and provides a simpler API.
     *
     * @author kazurayam
     */
    public class DockerBackedWebDriverTest {

        private static final int HOST_PORT = 3080;

        private static final PublishedPort publishedPort = new PublishedPort(HOST_PORT, 8080);
        private static final DockerImage image = new DockerImage("kazurayam/flaskr-kazurayam:1.1.0");

        private WebDriver driver = null;

        /**
         * start a Docker Container by "docker run" command.
         * In the container, a web server application runs to server a URL http://127.0.0.1:3080/
         *
         * It takes a bit long time; approximately 5 seconds. Just wait!
         */
        @BeforeAll
        public static void beforeAll() throws IOException, InterruptedException {
            File directory = Files.createTempDirectory("DockerBackedWebDriverTest").toFile();
            ContainerRunningResult crr =
                    ContainerRunner.runContainerAtHostPort(directory, publishedPort, image);
            if (crr.returncode() != 0) {
                throw new IllegalStateException(crr.toString());
            }
            // setup ChromeDriver
            WebDriverManager.chromedriver().setup();
        }

        /**
         * open a Chrome browser window
         */
        @BeforeEach
        public void beforeEach() {
            driver = new ChromeDriver();
        }

        /**
         * Test an HTML page.
         * Will verify if the site name in the page header is "Flaskr".
         */
        @Test
        public void test_page_header() {
            driver.navigate().to(String.format("http://127.0.0.1:%d/", HOST_PORT));
            WebElement siteName = driver.findElement(By.xpath("/html/body/nav/h1"));
            assertNotNull(siteName);
            assertEquals("Flaskr", siteName.getText());
            delay(2000);
        }

        /**
         * close the Chrome browser window
         */
        @AfterEach
        public void afterEach() {
            if (driver != null) {
                driver.quit();
                driver = null;
            }
        }


        /**
         * Stop the Docker Container gracefully by the "docker stop" command.
         * It will take approximately 10 seconds.
         * Be tolerant. Just wait!
         */
        @AfterAll
        public static void afterAll() throws IOException, InterruptedException {
            ContainerFindingResult cfr = ContainerFinder.findContainerByHostPort(HOST_PORT);
            if (cfr.returncode() == 0) {
                ContainerId containerId = cfr.containerId();
                ContainerStoppingResult csr = ContainerStopper.stopContainer(containerId);
                if (csr.returncode() != 0) {
                    throw new IllegalStateException(csr.toString());
                }
            } else {
                throw new IllegalStateException(cfr.toString());
            }
        }

        private void printResult(String label, ContainerFindingResult cfr) {
            System.out.println("-------- " + label + " --------");
            System.out.println(cfr.toString());
        }

        private void delay(int millis) {
            try {
                long l = (long)millis;
                Thread.sleep(l);
            } catch(InterruptedException e) {
                e.printStackTrace();
            }
        }

    }

## How to reuse this

See the [build.gradle](https://github.com/kazurayam/SeleniumTestInJavaBackedByDockerContainer/blob/master/build.gradle)

    /**
     * Selenium Test in Java backed by Docker Container
     */

    plugins {
        id 'java'
        id 'idea'
    }

    repositories {
        mavenCentral()
        mavenLocal()
    }

    group = 'com.kazurayam'
    archivesBaseName = "example"
    version = '0.1.0'

    ext.isReleaseVersion = ! version.endsWith("SNAPSHOT")

    sourceCompatibility = '1.8'
    targetCompatibility = '1.8'

    def defaultEncoding = 'UTF-8'
    tasks.withType(AbstractCompile).each { it.options.encoding = defaultEncoding }

    dependencies {
        // https://mvnrepository.com/artifact/com.kazurayam/subprocessj
        implementation group: 'com.kazurayam', name: 'subprocessj', version: '0.3.0'

        testImplementation 'org.junit.jupiter:junit-jupiter-api:5.8.2'
        testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.8.2'

        // https://mvnrepository.com/artifact/org.seleniumhq.selenium/selenium-java
        testImplementation group: 'org.seleniumhq.selenium', name: 'selenium-java', version: '4.1.2'

        // https://mvnrepository.com/artifact/io.github.bonigarcia/webdrivermanager
        testImplementation group: 'io.github.bonigarcia', name: 'webdrivermanager', version: '5.0.3'
        testImplementation 'org.slf4j:slf4j-api:1.7.35'
        testImplementation 'org.slf4j:slf4j-simple:1.7.35'
    }

    test {
        useJUnitPlatform()
    }

## Conclusion

The [Subprocessj 0.3.0](https://mvnrepository.com/artifact/com.kazurayam/subprocessj) library enabled me to perform automated Web UI testings written in Java/Groovy against a web application written in Python in Docker Container. I could use Java and other programming languages mixed and integrated to build my applications. I am contented with the Subprocessj.

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
