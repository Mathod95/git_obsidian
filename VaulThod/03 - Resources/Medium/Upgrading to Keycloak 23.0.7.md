---
tags:
  - APP/KEYCLOAK
source: https://tech-talk.the-experts.nl/upgrading-to-keycloak-23-0-2-324af9da071a
---




# Upgrading to Keycloak 23.0.7



## Keycloak Configuration as Code Pt. 5

![](https://miro.medium.com/v2/resize:fit:700/0*C2RCpMXyPn5Kqiub.jpg) Leveraging the power of configuration as code with Keycloak as an IDP!
Keycloak is an open-source identity and access management solution that provides features such as Single Sign-On, social login, and user federation. It is widely used by organizations to secure their web applications and APIs.
One of the main advantages of Keycloak is its Java Admin Client, which allows developers to automate the management of Keycloak through Java code. The Java Admin Client provides a rich set of APIs that allow developers to perform a wide range of administrative tasks, such as creating and managing realms, users, roles, groups, and clients. In this hands-on blog series, we will work towards a fully automated configuration, exclusively using such Java code configurations. Whether you are a Keycloak beginner or an experienced developer, this blog series will provide valuable insights into the configuration of Keycloak using Java code and how it can make your life easier.
What we have seen so far:
1.   [Identity and Access Management with Keycloak](https://medium.com/the-experts-tech-talks/identity-and-access-management-iam-with-keycloak-a2b601bcd34e)  — In this blog post, we get to know some basic building blocks that we have at our disposal in Keycloak.
2.   [Keycloak — Configuration as Code Pt. 1](https://medium.com/the-experts-tech-talks/keycloak-configuration-as-code-pt-1-c143a98a0165)  — In this blog post, we cover how to create the basic project setup for our ‘ *Keycloak — Configuration as Code* ’ endeavor.
3.   [Keycloak — Configuration as Code Pt. 2](https://medium.com/the-experts-tech-talks/keycloak-configuration-as-code-pt-2-d9ce38c1a7f7)  — In this blog post, we add the Keycloak distribution to our project and containerize it with Docker.
4.   [Keycloak — Configuration as Code Pt. 3](https://medium.com/the-experts-tech-talks/keycloak-configuration-as-code-pt-3-75392d06bca0)  — In this blog post, we kicked off the configuration as code by adding a first example realm to our Keycloak instance.
5.   [Upgrading to Keycloak 22.0.5 — To dependency hell and back](https://medium.com/the-experts-tech-talks/upgrading-to-keycloak-22-0-5-to-dependency-hell-and-back-68b75396563e)  — In this blog post, we tackled the not so out of the box upgrade from Keycloak 21 to 22.
6.   [Keycloak — Configuration as Code Pt. 4 — You build it, you run it (and test it)!](https://tech-talk.the-experts.nl/keycloak-configuration-as-code-pt-4-c1dc29601c54)  — In this blog post, we take steps towards a buildable and runnable setup and add unit test coverage to our project.

We have only recently upgraded our Keycloak instance in a painful process to version 22.0.5. Shortly after, Keycloak 23 was released, which comes with an important fix for  [CVE 2023–6291](https://www.cvedetails.com/cve/CVE-2023-6291/) 
> 
 **CVE 2023–6291 Description** 
“A flaw was found in the redirect_uri validation logic in Keycloak. This issue may allow a bypass of otherwise explicitly allowed hosts. A successful attack may lead to an access token being stolen, making it possible for the attacker to impersonate other users.”

Unfortunately, the first few minor versions of Keycloak 23 were shipped with a  [major bug](https://github.com/keycloak/keycloak/issues/25010) , that prevented us from correctly processing the  `KC_DB_USERNAME` variable.  [This was fixed](https://github.com/keycloak/keycloak/pull/25088)  and released in Keycloak 23.0.4. Hence, the advice to not use one of the earlier minor release versions < 23.0.4 if you depend on an external DB setup.


##  *Disclaimer* 

 *It may be the case that you are running Keycloak in a productive environment, and you cannot simply throw away the DB and relaunch it with the newer Keycloak version, as we do in this tutorial. Keycloak provides you with *  [ *upgrade instructions*  ](https://www.keycloak.org/docs/latest/upgrading/index.html#manual-relational-database-migration) * and how to achieve a (manual) database migration, if required.* 


## Status Quo

In part 5 of this tutorial series,  [Upgrading to Keycloak 22.0.5 — To dependency hell and back](https://medium.com/the-experts-tech-talks/upgrading-to-keycloak-22-0-5-to-dependency-hell-and-back-68b75396563e) , we have successfully altered our setup through a tedious process to run Keycloak 22.0.5. Due to the discovered CVE mentioned above, it is time to upgrade to Keycloak 23.0.6.
During that process, we have altered and / or excluded a number of dependencies that caused a variety of errors in our setup. In order to be able to validate, if these alterations are still required, we, as a first step, will comment all of them out and raise the version of Keycloak to named 23.0.6. We also raise the Quarkus version to  `3.2.10.Final` , as Keycloak uses that one itself. This leaves us with the following  `./pom.xml` 

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>nl.the_experts.keycloak</groupId>
    <artifactId>keycloak-configascode-demo</artifactId>
    <version>0.0.1-local</version>
    <packaging>pom</packaging>

    <modules>
        <module>keycloak</module>
        <module>java-configuration</module>
    </modules>

    <properties>
        <!-- Java -->
        <java.version>17</java.version>

        <!-- Maven -->
        <maven-surefire-plugin.version>3.2.2</maven-surefire-plugin.version>
        <maven.shade.plugin.version>3.4.1</maven.shade.plugin.version>
        <maven.jar.plugin.version>3.3.0</maven.jar.plugin.version>
        <maven.dependency.plugin.version>3.6.0</maven.dependency.plugin.version>
        <maven-compiler-plugin.version>3.11.0</maven-compiler-plugin.version>
        <maven.resources.plugin.version>3.3.1</maven.resources.plugin.version>
        <maven.surefire.tree.reporter>1.2.1</maven.surefire.tree.reporter>

        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <maven.build.timestamp.format>yyyy-MM-dd'T'HH:mm:ss'Z'</maven.build.timestamp.format>

        <!-- Project specific -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <!-- Keycloak -->
        <keycloak.version>23.0.6</keycloak.version>
        <quarkus.version>3.2.10.Final</quarkus.version>
        <quarkus.native.builder-image>mutable-jar</quarkus.native.builder-image>
        <jandex.version>3.1.5</jandex.version>
        <asm.version>9.5</asm.version>
        <resteasy.version>6.2.4.Final</resteasy.version>

        <!-- Plugins -->
        <build-helper-maven-plugin.version>3.3.0</build-helper-maven-plugin.version>

        <!-- Tooling -->
        <lombok.version>1.18.30</lombok.version>

        <!-- Testing -->
        <junit.jupiter.version>5.10.1</junit.jupiter.version>
        <assertj-core.version>3.24.2</assertj-core.version>
        <mockito.version>5.8.0</mockito.version>
        <junit.pioneer.version>2.2.0</junit.pioneer.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!--Quarkus-->
<!--            <dependency>-->
<!--                <groupId>io.quarkus.arc</groupId>-->
<!--                <artifactId>arc</artifactId>-->
<!--                <version>${quarkus.version}</version>-->
<!--            </dependency>-->

            <!--Keycloak-->
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-quarkus-server</artifactId>
                <version>${keycloak.version}</version>
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-core</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss</groupId>-->
<!--                        <artifactId>jandex</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.ow2.asm</groupId>-->
<!--                        <artifactId>asm</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.ow2.asm</groupId>-->
<!--                        <artifactId>asm-commons</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.ow2.asm</groupId>-->
<!--                        <artifactId>asm-tree</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.ow2.asm</groupId>-->
<!--                        <artifactId>asm-util</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
            </dependency>
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-quarkus-dist</artifactId>
                <version>${keycloak.version}</version>
                <type>zip</type>
            </dependency>
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-admin-client</artifactId>
                <version>${keycloak.version}</version>
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-client</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-jackson2-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-jaxb-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
            </dependency>
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-core</artifactId>
                <version>${keycloak.version}</version>
            </dependency>
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-services</artifactId>
                <version>${keycloak.version}</version>
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
            </dependency>

            <!--Fix versions-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-core</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss</groupId>-->
<!--                        <artifactId>jandex</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-client</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-jackson2-provider</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.ow2.asm</groupId>-->
<!--                <artifactId>asm</artifactId>-->
<!--                <version>${asm.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.ow2.asm</groupId>-->
<!--                <artifactId>asm-commons</artifactId>-->
<!--                <version>${asm.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.ow2.asm</groupId>-->
<!--                <artifactId>asm-util</artifactId>-->
<!--                <version>${asm.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.ow2.asm</groupId>-->
<!--                <artifactId>asm-tree</artifactId>-->
<!--                <version>${asm.version}</version>-->
<!--            </dependency>-->

            <!--Tooling-->
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </dependency>

            <!--Test-->
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter-engine</artifactId>
                <version>${junit.jupiter.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter-params</artifactId>
                <version>${junit.jupiter.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter</artifactId>
                <version>${junit.jupiter.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.assertj</groupId>
                <artifactId>assertj-core</artifactId>
                <version>${assertj-core.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.mockito</groupId>
                <artifactId>mockito-core</artifactId>
                <version>${mockito.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.mockito</groupId>
                <artifactId>mockito-junit-jupiter</artifactId>
                <version>${mockito.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.junit-pioneer</groupId>
                <artifactId>junit-pioneer</artifactId>
                <version>${junit.pioneer.version}</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
      .....
    </build>
</project>
```


Since we removed the fixed dependencies from our dependency management in the parent pom, make sure to remove them from the child pom files as well. With these adaptions in place, we will try to build our project, by running:

```
mvn clean package
```


This will result in a well known error we have seen in our previous attempt to upgrade Keycloak:

```
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal io.quarkus:quarkus-maven-plugin:3.2.7.Final:build 
        (default) on project keycloak: Execution default of goal 
        io.quarkus:quarkus-maven-plugin:3.2.7.Final:build failed:
        An API incompatibility was encountered while executing 
        io.quarkus:quarkus-maven-plugin:3.2.7.Final:build: 
        java.lang.NoSuchMethodError:
        'org.jboss.jandex.DotName org.jboss.jandex.DotName.createSimple(java.lang.Class)'
[ERROR] -----------------------------------------------------
```


Hence, we re-add the exclusion of the  `jandex` dependency in the dependency management of our parent pom.xml file, as it still causes the same error.
Doing so, and re-executing the  `mvn clean package`  command, will lead to the next well known error:

```
[ERROR] Failed to execute goal io.quarkus:quarkus-maven-plugin:3.2.10.Final:build 
(default) on project keycloak: Failed to build quarkus application: 
io.quarkus.builder.BuildException: Build failure: Build failed due to errors
<...>
[ERROR] Caused by: java.lang.IllegalArgumentException: Unsupported api 589824
[ERROR]         at org.objectweb.asm.ClassVisitor.<init>(ClassVisitor.java:74)
[ERROR]         at org.objectweb.asm.ClassVisitor.<init>(ClassVisitor.java:57)
```


So, once again, we re-add all the  `` related exclusions in our dependency management and fixing the correct versions hereafter. We also need to re-add the explicit  ``  dependencies in the  `/keycloak/pom.xml` . That leaves us with the following:
./pom.xml:

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>nl.the_experts.keycloak</groupId>
    <artifactId>keycloak-configascode-demo</artifactId>
    <version>0.0.1-local</version>
    <packaging>pom</packaging>

    <modules>
        <module>keycloak</module>
        <module>java-configuration</module>
    </modules>

    <properties>
        <!-- Java -->
        <java.version>17</java.version>

        <!-- Maven -->
        <maven-surefire-plugin.version>3.2.2</maven-surefire-plugin.version>
        <maven.shade.plugin.version>3.4.1</maven.shade.plugin.version>
        <maven.jar.plugin.version>3.3.0</maven.jar.plugin.version>
        <maven.dependency.plugin.version>3.6.0</maven.dependency.plugin.version>
        <maven-compiler-plugin.version>3.11.0</maven-compiler-plugin.version>
        <maven.resources.plugin.version>3.3.1</maven.resources.plugin.version>
        <maven.surefire.tree.reporter>1.2.1</maven.surefire.tree.reporter>

        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <maven.build.timestamp.format>yyyy-MM-dd'T'HH:mm:ss'Z'</maven.build.timestamp.format>

        <!-- Project specific -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <!-- Keycloak -->
        <keycloak.version>23.0.6</keycloak.version>
        <quarkus.version>3.2.10.Final</quarkus.version>
        <quarkus.native.builder-image>mutable-jar</quarkus.native.builder-image>
        <jandex.version>3.1.5</jandex.version>
        <asm.version>9.5</asm.version>
        <resteasy.version>6.2.4.Final</resteasy.version>

        <!-- Plugins -->
        <build-helper-maven-plugin.version>3.3.0</build-helper-maven-plugin.version>

        <!-- Tooling -->
        <lombok.version>1.18.30</lombok.version>

        <!-- Testing -->
        <junit.jupiter.version>5.10.1</junit.jupiter.version>
        <assertj-core.version>3.24.2</assertj-core.version>
        <mockito.version>5.8.0</mockito.version>
        <junit.pioneer.version>2.2.0</junit.pioneer.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!--Quarkus-->
<!--            <dependency>-->
<!--                <groupId>io.quarkus.arc</groupId>-->
<!--                <artifactId>arc</artifactId>-->
<!--                <version>${quarkus.version}</version>-->
<!--            </dependency>-->

            <!--Keycloak-->
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-quarkus-server</artifactId>
                <version>${keycloak.version}</version>
                <exclusions>
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-core</artifactId>-->
<!--                    </exclusion>-->
                    <exclusion>
                        <groupId>org.jboss</groupId>
                        <artifactId>jandex</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.ow2.asm</groupId>
                        <artifactId>asm</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.ow2.asm</groupId>
                        <artifactId>asm-commons</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.ow2.asm</groupId>
                        <artifactId>asm-tree</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.ow2.asm</groupId>
                        <artifactId>asm-util</artifactId>
                    </exclusion>
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
                </exclusions>
            </dependency>
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-quarkus-dist</artifactId>
                <version>${keycloak.version}</version>
                <type>zip</type>
            </dependency>
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-admin-client</artifactId>
                <version>${keycloak.version}</version>
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-client</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-jackson2-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-jaxb-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
            </dependency>
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-core</artifactId>
                <version>${keycloak.version}</version>
            </dependency>
            <dependency>
                <groupId>org.keycloak</groupId>
                <artifactId>keycloak-services</artifactId>
                <version>${keycloak.version}</version>
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
            </dependency>

            <!--Fix versions-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-core</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss</groupId>-->
<!--                        <artifactId>jandex</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-client</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-jackson2-provider</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
            <dependency>
                <groupId>org.ow2.asm</groupId>
                <artifactId>asm</artifactId>
                <version>${asm.version}</version>
            </dependency>
            <dependency>
                <groupId>org.ow2.asm</groupId>
                <artifactId>asm-commons</artifactId>
                <version>${asm.version}</version>
            </dependency>
            <dependency>
                <groupId>org.ow2.asm</groupId>
                <artifactId>asm-util</artifactId>
                <version>${asm.version}</version>
            </dependency>
            <dependency>
                <groupId>org.ow2.asm</groupId>
                <artifactId>asm-tree</artifactId>
                <version>${asm.version}</version>
            </dependency>

            <!--Tooling-->
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </dependency>

            <!--Test-->
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter-engine</artifactId>
                <version>${junit.jupiter.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter-params</artifactId>
                <version>${junit.jupiter.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter</artifactId>
                <version>${junit.jupiter.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.assertj</groupId>
                <artifactId>assertj-core</artifactId>
                <version>${assertj-core.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.mockito</groupId>
                <artifactId>mockito-core</artifactId>
                <version>${mockito.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.mockito</groupId>
                <artifactId>mockito-junit-jupiter</artifactId>
                <version>${mockito.version}</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.junit-pioneer</groupId>
                <artifactId>junit-pioneer</artifactId>
                <version>${junit.pioneer.version}</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>${maven-compiler-plugin.version}</version>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>${maven.resources.plugin.version}</version>
                </plugin>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>${maven-surefire-plugin.version}</version>
                    <dependencies>
                        <dependency>
                            <groupId>org.junit.jupiter</groupId>
                            <artifactId>junit-jupiter-engine</artifactId>
                            <version>${junit.jupiter.version}</version>
                        </dependency>
                        <dependency>
                            <groupId>me.fabriciorby</groupId>
                            <artifactId>maven-surefire-junit5-tree-reporter</artifactId>
                            <version>${maven.surefire.tree.reporter}</version>
                        </dependency>
                    </dependencies>
                    <configuration>
                        <excludes>
                            <exclude>**/*IntegrationTest.java</exclude>
                        </excludes>
                        <reportFormat>plain</reportFormat>
                        <consoleOutputReporter>
                            <disable>true</disable>
                        </consoleOutputReporter>
                        <statelessTestsetInfoReporter implementation="org.apache.maven.plugin.surefire.extensions.junit5.JUnit5StatelessTestsetInfoTreeReporter"/>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-dependency-plugin</artifactId>
                    <version>${maven.dependency.plugin.version}</version>
                    <executions>
                        <execution>
                            <id>unpack-keycloak-server-distribution</id>
                            <phase>package</phase>
                            <goals>
                                <goal>unpack</goal>
                            </goals>
                            <configuration>
                                <artifactItems>
                                    <artifactItem>
                                        <groupId>org.keycloak</groupId>
                                        <artifactId>keycloak-quarkus-dist</artifactId>
                                        <type>zip</type>
                                        <outputDirectory>target</outputDirectory>
                                    </artifactItem>
                                </artifactItems>
                                <excludes>**/lib/**</excludes>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>io.quarkus</groupId>
                    <artifactId>quarkus-maven-plugin</artifactId>
                    <version>${quarkus.version}</version>
                    <configuration>
                        <finalName>keycloak</finalName>
                        <buildDir>${project.build.directory}/keycloak-${keycloak.version}</buildDir>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>build</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```


 `/keycloak/pom.xml` 

```
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>nl.the_experts.keycloak</groupId>
        <artifactId>keycloak-configascode-demo</artifactId>
        <version>0.0.1-local</version>
    </parent>
    <artifactId>keycloak</artifactId>
    <dependencies>
<!--        <dependency>-->
<!--            <groupId>io.quarkus.arc</groupId>-->
<!--            <artifactId>arc</artifactId>-->
<!--        </dependency>-->
        <dependency>
            <!-- Keycloak Distribution -->
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-quarkus-dist</artifactId>
            <type>zip</type>
        </dependency>
        <dependency>
            <!-- Keycloak Quarkus Server Libraries-->
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-quarkus-server</artifactId>
        </dependency>

<!--        Fix versions-->
<!--        <dependency>-->
<!--            <groupId>org.jboss.resteasy</groupId>-->
<!--            <artifactId>resteasy-core</artifactId>-->
<!--        </dependency>-->
<!--        <dependency>-->
<!--            <groupId>org.jboss.resteasy</groupId>-->
<!--            <artifactId>resteasy-client</artifactId>-->
<!--        </dependency>-->
<!--        <dependency>-->
<!--            <groupId>org.jboss.resteasy</groupId>-->
<!--            <artifactId>resteasy-jackson2-provider</artifactId>-->
<!--        </dependency>-->
<!--        <dependency>-->
<!--            <groupId>org.jboss.resteasy</groupId>-->
<!--            <artifactId>resteasy-multipart-provider</artifactId>-->
<!--        </dependency>-->
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm</artifactId>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-commons</artifactId>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-util</artifactId>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-tree</artifactId>
        </dependency>
    </dependencies>

    <build>
        <finalName>keycloak-${project.version}</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```


We build our project once more, by running  `mvn clean package` , leaving us with a successful build.
![](https://miro.medium.com/v2/resize:fit:681/1*Yxvo7HfIESO8EWGjwa-Tvg.png) 
Examining the generated  `/keycloak/target/`  folder, we also see that the folder containing our Keycloak server is now obviously called  `keycloak-23.0.6` . Hence, we update the build section of our  `./dockerfile`  file as follows:

```
FROM registry.access.redhat.com/ubi8-minimal:8.9 AS builder
RUN microdnf update -y && \
    microdnf install -y java-17-openjdk-headless && microdnf clean all && rm -rf /var/cache/yum/* && \
    echo "keycloak:x:0:root" >> /etc/group && \
    echo "keycloak:x:1000:0:keycloak user:/opt/keycloak:/sbin/nologin" >> /etc/passwd

COPY --chown=keycloak:keycloak keycloak/target/keycloak-23.0.6  /opt/keycloak

USER 1000

RUN /opt/keycloak/bin/kc.sh build --db=postgres
```


Last but not least, with the release of Keycloak 23, the already deprecated  `--auto-build`  CLI option  [was removed from Keycloak](https://www.keycloak.org/docs/latest/upgrading/index.html#the-deprecated-auto-build-cli-option-was-removed) . We need to update the entry point of our Keycloak container in the  `./docker-compose.yml`  as follows:

```
entrypoint: /opt/keycloak/bin/kc.sh start-dev --http-enabled=true --cache=local
```


Given the previous successful build execution, we now try to launch our setup by executing the command  `docker compose up --build --force-recreate`  in our shell.
That leaves us with a new error we haven’t seen before. During the launch of the Keycloak container, we see the following log:

```
Uncaught exception received by Vert.x: java.lang.NoClassDefFoundError: io/vertx/core/net/HostAndPort
<...>
Caused by: java.lang.ClassNotFoundException: io.vertx.core.net.HostAndPort
       at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:641)
       at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:188)
       at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:525)
       at io.quarkus.bootstrap.runner.RunnerClassLoader.loadClass(RunnerClassLoader.java:115)
       at io.quarkus.bootstrap.runner.RunnerClassLoader.loadClass(RunnerClassLoader.java:65)
       ... 52 more

```




## Vert.x

Quarkus integrates Vert.x as one of its core components. This integration allows Quarkus applications to leverage the reactive programming model and the non-blocking, event-driven capabilities of Vert.x, making it easier to build efficient and scalable microservices and web applications. The combination of Quarkus and Vert.x offers developers the tools to create applications that can handle high concurrency with low latency, making it suitable for modern, cloud-native applications that demand efficient resource utilization and fast response times.
During the launch of Keycloak, we can actually see the following log statement, one of the installed features being  `vertx` 

```
2024-03-01 14:11:46,453 INFO  [io.quarkus] (main) Installed features: 
[agroal, cdi, hibernate-orm, jdbc-h2, jdbc-mariadb, jdbc-mssql, jdbc-mysql, jdbc-oracle, jdbc-postgresql, keycloak, logging-gelf, micrometer, narayana-jta, reactive-routes, resteasy-reactive, resteasy-reactive-jackson, smallrye-context-propagation, smallrye-health, vertx]
```


 [The configuration option](https://www.keycloak.org/server/all-config)  `feartures-disabled`  does not allow for  `vertx`  to be disabled. Let's take a deeper look into where this dependency actually comes from:
![](https://miro.medium.com/v2/resize:fit:700/1*mr2yJvJVaLa1tpB_CYi3bg.png) Source of the Quarkus vert.x dependency

![](https://miro.medium.com/v2/resize:fit:700/1*DX_3QYINQJY19_rPfnK-Ow.png) Source of the Quarkus vert.x HTTP dependency
The error at hand states that a class cannot be found from the vert.x core. Analyzing that dependency, we quickly see that there is a version conflict in the transitive dependencies concerning  `vertx-core`  `quarkus-vertx`  expects, ⁣ but `vertx-core:4.4.6`  `quarkus-vertx-http`  loads  `vertx-core:4.3.8`  as a transitive dependency.
![](https://miro.medium.com/v2/resize:fit:700/1*ce0OSDF6EM9tal4IMJ-5Hg.png) Vert.x core transitive dependency version clash
Knowing where the  `quarkus-vertx`  dependencies come from and which version we need for  `vertx-core` , let us start to fix this error in our dependency management:

```
<dependencyManagement>
    <dependencies>
        <!--Quarkus-->
<!--            <dependency>-->
<!--                <groupId>io.quarkus.arc</groupId>-->
<!--                <artifactId>arc</artifactId>-->
<!--                <version>${quarkus.version}</version>-->
<!--            </dependency>-->

        <!--Keycloak-->
        <dependency>
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-quarkus-server</artifactId>
            <version>${keycloak.version}</version>
            <exclusions>
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-core</artifactId>-->
<!--                    </exclusion>-->
                <exclusion>
                    <groupId>org.jboss</groupId>
                    <artifactId>jandex</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.ow2.asm</groupId>
                    <artifactId>asm</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.ow2.asm</groupId>
                    <artifactId>asm-commons</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.ow2.asm</groupId>
                    <artifactId>asm-tree</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.ow2.asm</groupId>
                    <artifactId>asm-util</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>io.quarkus</groupId>
                    <artifactId>quarkus-vertx</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>io.quarkus</groupId>
                    <artifactId>quarkus-vertx-http</artifactId>
                </exclusion>
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-quarkus-dist</artifactId>
            <version>${keycloak.version}</version>
            <type>zip</type>
        </dependency>
        <dependency>
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-admin-client</artifactId>
            <version>${keycloak.version}</version>
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-client</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-jackson2-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-jaxb-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
        </dependency>
        <dependency>
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-core</artifactId>
            <version>${keycloak.version}</version>
        </dependency>
        <dependency>
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-services</artifactId>
            <version>${keycloak.version}</version>
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss.resteasy</groupId>-->
<!--                        <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
        </dependency>

        <!--Fix versions-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-core</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--                <exclusions>-->
<!--                    <exclusion>-->
<!--                        <groupId>org.jboss</groupId>-->
<!--                        <artifactId>jandex</artifactId>-->
<!--                    </exclusion>-->
<!--                </exclusions>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-client</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-jackson2-provider</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
<!--            <dependency>-->
<!--                <groupId>org.jboss.resteasy</groupId>-->
<!--                <artifactId>resteasy-multipart-provider</artifactId>-->
<!--                <version>${resteasy.version}</version>-->
<!--            </dependency>-->
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm</artifactId>
            <version>${asm.version}</version>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-commons</artifactId>
            <version>${asm.version}</version>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-util</artifactId>
            <version>${asm.version}</version>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-tree</artifactId>
            <version>${asm.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-vertx</artifactId>
            <version>${quarkus.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>io.vertx</groupId>
                    <artifactId>vertx-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-vertx-http</artifactId>
            <version>${quarkus.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>io.vertx</groupId>
                    <artifactId>vertx-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.vertx</groupId>
            <artifactId>vertx-core</artifactId>
            <version>4.4.6</version>
        </dependency>

        <!--Tooling-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
        </dependency>

        <!--Test-->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>${junit.jupiter.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-params</artifactId>
            <version>${junit.jupiter.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.jupiter.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>${assertj-core.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>${mockito.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-junit-jupiter</artifactId>
            <version>${mockito.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit-pioneer</groupId>
            <artifactId>junit-pioneer</artifactId>
            <version>${junit.pioneer.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```


and the following dependencies in  `/keycloak/pom.xml` 

```
    <dependencies>
<!--        <dependency>-->
<!--            <groupId>io.quarkus.arc</groupId>-->
<!--            <artifactId>arc</artifactId>-->
<!--        </dependency>-->
        <dependency>
            <!-- Keycloak Distribution -->
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-quarkus-dist</artifactId>
            <type>zip</type>
        </dependency>
        <dependency>
            <!-- Keycloak Quarkus Server Libraries-->
            <groupId>org.keycloak</groupId>
            <artifactId>keycloak-quarkus-server</artifactId>
        </dependency>

<!--        Fix versions-->
<!--        <dependency>-->
<!--            <groupId>org.jboss.resteasy</groupId>-->
<!--            <artifactId>resteasy-core</artifactId>-->
<!--        </dependency>-->
<!--        <dependency>-->
<!--            <groupId>org.jboss.resteasy</groupId>-->
<!--            <artifactId>resteasy-client</artifactId>-->
<!--        </dependency>-->
<!--        <dependency>-->
<!--            <groupId>org.jboss.resteasy</groupId>-->
<!--            <artifactId>resteasy-jackson2-provider</artifactId>-->
<!--        </dependency>-->
<!--        <dependency>-->
<!--            <groupId>org.jboss.resteasy</groupId>-->
<!--            <artifactId>resteasy-multipart-provider</artifactId>-->
<!--        </dependency>-->
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm</artifactId>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-commons</artifactId>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-util</artifactId>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-tree</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-vertx</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-vertx-http</artifactId>
        </dependency>
        <dependency>
            <groupId>io.vertx</groupId>
            <artifactId>vertx-core</artifactId>
        </dependency>
    </dependencies>
```


Having done so, we now repackage our application with  `mvn clean package`  and then restart our docker compose containers by running  `docker compose down && docker compose up --build --force-recreate` 
The docker container logs of our  `java-configuration`  container now show us the following error:

```
<... Partial extract from logs ...>

Caused by: jakarta.ws.rs.ProcessingException: RESTEASY003215: could not find writer for content-type application/x-www-form-urlencoded type: jakarta.ws.rs.core.Form$1
    at org.jboss.resteasy.core.interception.jaxrs.ClientWriterInterceptorContext.throwWriterNotFoundException(ClientWriterInterceptorContext.java:48)

<...>
```


Yet another error well known to us from one of the previous blogs:  [Upgrading to Keycloak 22.0.5 — To dependency hell and back](https://medium.com/the-experts-tech-talks/upgrading-to-keycloak-22-0-5-to-dependency-hell-and-back-68b75396563e) 
> 
“We are STILL missing the resteasy-multipart-provider!”

This is a weird case of inexplicable wrong dependency resolution (at least I could not find the cause yet). Although the maven tree shows that the  `resteasy-multipart-provider`  is available, it is not available at runtime when we try to execute the java configuration.
We have fixed this in previous versions and will do so again by excluding resteasy related dependencies from the  `keycloak-quarkus-server`  dependency in our room  `pom.xml` 

```
<...>  

  <groupId>org.keycloak</groupId>
  <artifactId>keycloak-quarkus-server</artifactId>
  <version>${keycloak.version}</version>
  <exclusions>
      <exclusion>
          <groupId>org.jboss.resteasy</groupId>
          <artifactId>resteasy-core</artifactId>
      </exclusion>
      <exclusion>
          <groupId>org.jboss.resteasy</groupId>
          <artifactId>resteasy-multipart-provider</artifactId>
      </exclusion>

      <...>

  </exclusions>

<...>
```


and similarly from the  `keycloak-admin-client`  and  `keylcoak-services` 

```
  <dependency>
      <groupId>org.keycloak</groupId>
      <artifactId>keycloak-admin-client</artifactId>
      <version>${keycloak.version}</version>
      <exclusions>
          <exclusion>
              <groupId>org.jboss.resteasy</groupId>
              <artifactId>resteasy-client</artifactId>
          </exclusion>
          <exclusion>
              <groupId>org.jboss.resteasy</groupId>
              <artifactId>resteasy-jackson2-provider</artifactId>
          </exclusion>
          <exclusion>
              <groupId>org.jboss.resteasy</groupId>
              <artifactId>resteasy-multipart-provider</artifactId>
          </exclusion>
          <exclusion>
              <groupId>org.jboss.resteasy</groupId>
              <artifactId>resteasy-jaxb-provider</artifactId>
          </exclusion>
      </exclusions>
  </dependency>
  <dependency>
      <groupId>org.keycloak</groupId>
      <artifactId>keycloak-services</artifactId>
      <version>${keycloak.version}</version>
      <exclusions>
          <exclusion>
              <groupId>org.jboss.resteasy</groupId>
              <artifactId>resteasy-multipart-provider</artifactId>
          </exclusion>
      </exclusions>
  </dependency>
```


We still do need them, though, hence we re-add them in our dependency-management block as follows:

```
<dependencyManagement>
    <dependencies>
<!--...-->

  <!--Fix versions-->
      <!--Resteasy-->
      <dependency>
          <groupId>org.jboss.resteasy</groupId>
          <artifactId>resteasy-core</artifactId>
          <version>${resteasy.version}</version>
          <exclusions>
              <exclusion>
                  <groupId>org.jboss</groupId>
                  <artifactId>jandex</artifactId>
              </exclusion>
          </exclusions>
      </dependency>
      <dependency>
          <groupId>org.jboss.resteasy</groupId>
          <artifactId>resteasy-client</artifactId>
          <version>${resteasy.version}</version>
      </dependency>
      <dependency>
          <groupId>org.jboss.resteasy</groupId>
          <artifactId>resteasy-jackson2-provider</artifactId>
          <version>${resteasy.version}</version>
      </dependency>
      <dependency>
          <groupId>org.jboss.resteasy</groupId>
          <artifactId>resteasy-multipart-provider</artifactId>
          <version>${resteasy.version}</version>
          <exclusions>
              <exclusion>
                  <!--CVE fix-->
                  <groupId>org.apache.james</groupId>
                  <artifactId>apache-mime4j-dom</artifactId>
              </exclusion>
          </exclusions>
      </dependency>
      <dependency>
          <groupId>org.apache.james</groupId>
          <artifactId>apache-mime4j-dom</artifactId>
          <version>0.8.11</version>
      </dependency>
      <dependency>
          <groupId>org.jboss.resteasy</groupId>
          <artifactId>resteasy-jaxb-provider</artifactId>
          <version>${resteasy.version}</version>
      </dependency>
    <!--...-->
   </dependencies>
</dependencyManagement>

```


Since they have been excluded from dependencies that are being used in our child modules, we equally re-add those dependencies needed in their dependencies respectively. In  `./keycloak-server/pom.xml`  we append:

```
    <!--Fix versions-->
        <!--Resteasy-->
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-multipart-provider</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.james</groupId>
            <artifactId>apache-mime4j-dom</artifactId>
        </dependency>
```


And in  `./java-configuration/pom.xml`  we append:

```
    <!--fix versions-->
        <!--Resteasy-->
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-jackson2-provider</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-multipart-provider</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.james</groupId>
            <artifactId>apache-mime4j-dom</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-jaxb-provider</artifactId>
        </dependency>
```


Having done so, we now once more repackage our application with  `mvn clean package`  and then restart our docker compose containers by running  `docker compose down && docker compose up --build --force-recreate` 
E voilà! We have fixed our setup once more and have thus successfully migrated our setup to Keycloak 23.0.7. Again, please bear in mind that extra steps need to be taken for a database migration in a production environment.
As always, the code can be found  [here](https://github.com/MaikKingma/keycloak-configAsCode-demo/tree/Blog-6/upgrade-to-23-0-7) 
Or are you interested in how to set up your Keycloak instance in such a way that DB migrations become almost irrelevant to your project? Then get in touch! Read also more on our website,  [the-experts.nl](http://the-experts.nl/) 
Stay tuned for more tutorials on how to programmatically add clients, custom identity providers and much more.