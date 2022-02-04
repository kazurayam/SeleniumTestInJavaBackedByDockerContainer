# Selenium Test in Java backed by Docker Container

@author kazurayam
@date 4 Feb, 2022

## Problem to solve

I am interested in automated tests for Web UI using Selenium WebDriver in Java/Groovy. Also I am interested in developing Web Server applications in Python. So I want to execute UI tests written in Java against my Web Application written in Python. I can do it by the following manual operations.

\(1\) in a Terminal window on Mac, I will start a Docker container to launch a HTTP server by

    $ cd ~/tmp
    $ docker run -d -p 80:8080 kazurayam/flaskr-kazurayam:1.1.0

\(2\) I will open another Terminal window, and execute UI tests by:

    $ cd $projectDir
    $ gradle test
    ....

\(3\) when the test has finished, I will go back to the first terminal window. I will identify the id of the running docker container.

    $ docker ps --filter publish=80 --filter status=running -q
    fd5ad3b76b13

\(4\) I will stop the container gracefully by:

    $ docker stop fd5ad3b76b13

This procedure is not difficult. But it is cumbersome. I often forget terminating the previous docker container. When I try to start a container again, I get an error saying "the IP port is already in use", which makes me irritated.

I want to automate starting and stopping any Docker Container on the localhost by JUnit tests.

## Solution

I have developed a JUnit test which does the following:

-   The test visits a URL "http://127.0.0.1:3080/" using Selenium WebDriver.

-   The URL is served by a process on the localhost, in which a Docker Container runs using a docker image that I prepared.

-   The web app was developed in Python language by the Pallets project, is published at [flaskr tutorial](https://flask.palletsprojects.com/en/2.0.x/tutorial/)

-   This test automates running and stopping a Docker Container using the well-known commandline commands: `docker run`, `docker ps` and `docker stop`.

-   The [`com.kazurayam.subprocessj.docker.ContainerRunner`](https://github.com/kazurayam/subprocessj/blob/master/src/main/java/com/kazurayam/subprocessj/docker/ContainerRunner.java) class wraps the "docker run" command.

-   The [`com.kazurayam.subprocessj.docker.ContainerFinder`](https://github.com/kazurayam/subprocessj/blob/master/src/main/java/com/kazurayam/subprocessj/docker/ContainerFinder.java) class wraps the "docker ps" command.

-   The [`com.kazurayam.subprocessj.docker.ContainerStopper`](https://github.com/kazurayam/subprocessj/blob/master/src/main/java/com/kazurayam/subprocessj/docker/ContainerStopper.java) class wraps the "docker stop" command.

-   These classes internally call [`java.lang.ProcessBuilder`](https://www.baeldung.com/java-lang-processbuilder-api) to execute the `docker` command from Java.

-   The `ProcessBuilder` is wrapped by the [`com.kazurayam.subprocessj.Subprocess`](https://github.com/kazurayam/subprocessj/blob/master/src/main/java/com/kazurayam/subprocessj/Subprocess.java) class which provides a simplified API to run arbitrary OS commands.

## Description

### Sequence diagram

The following diagram shows how the sample code works.

![sequence](https://kazurayam.github.io/SeleniumTestInJavaBackedByDockerContainer/diagrams/out/sequence.png)

### Sample code

For full code, see the follow the link.

-   [example.DockerBackedSeleniumTest](https://github.com/kazurayam/SeleniumTestInJavaBackedByDockerContainer/blob/master/src/test/java/example/DockerBackedWebDriverTest.java)

I will quote from the source some fragments that would be of your interest.

#### @BeforeAll

starting a Docker container

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

#### @BeforeEach

opening a web browser

        @BeforeEach
        public void beforeEach() {
            driver = new ChromeDriver();
        }

#### @Test

navigate to a URL, verify the HTML

        @Test
        public void test_page_header() {
            driver.navigate().to(String.format("http://127.0.0.1:%d/", HOST_PORT));
            WebElement siteName = driver.findElement(By.xpath("/html/body/nav/h1"));
            assertNotNull(siteName);
            assertEquals("Flaskr", siteName.getText());
            delay(2000);
        }

The page looks like this:

![Flaskr](./docs/images/Flaskr.png)

#### @AfterEach

closing the browser

        @AfterEach
        public void afterEach() {
            if (driver != null) {
                driver.quit();
                driver = null;
            }
        }

#### @AfterAll

stopping the Docker container

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

## How to reuse this

See the [build.gradle](https://github.com/kazurayam/SeleniumTestInJavaBackedByDockerContainer/blob/master/build.gradle)

    /**
     * Selenium Test in Java backed by Docker Container
     */

    plugins {
        id 'java'
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
