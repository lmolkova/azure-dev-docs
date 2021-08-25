---
title: Troubleshoot dependency version conflict when using the Azure SDK for Java
description: An overview of how to troubleshoot dependency version conflicts related to using the Azure SDK for Java
ms.date: 08/24/2021
ms.topic: conceptual
ms.custom: devx-track-java
ms.author: lmolkova
---

# Troubleshooting Dependency Version Conflicts

Azure SDKs depend on several popular third-party libraries: [Jackson](https://github.com/FasterXML/jackson), [Netty](https://netty.io/), [Reactor](https://projectreactor.io/)
[slf4j](http://www.slf4j.org/), and a [few more](https://azure.github.io/azure-sdk/docs/java/approved_dependencies.html) that are SDk-specific.

Java applications and frameworks frequently use the same libraries directly or transitively, which leads to different components expecting different versions of the same library. Maven, Gradle and other package managers resolve conflicts at a build time using different strategies (check out [Maven](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html) and [Gradle](https://docs.gradle.org/current/userguide/dependency_resolution.html) dependency resolution documentation).
To resolve version conflict, package manager picks one version, so components may end up with incompatible depedency version.

Incomatibiliy on APIs level manifests as runtime exceptions [NoClassDefFoundError](https://docs.oracle.com/javase/8/docs/api/java/lang/NoClassDefFoundError.html), [NoSuchMethodError](https://docs.oracle.com/javase/8/docs/api/java/lang/NoSuchMethodError.html) or other [LinkageError](https://docs.oracle.com/javase/8/docs/api/java/lang/LinkageError.html). Note that not all libraries strictly follow [Semantic Versioning](https://semver.org/) and breaking changes sometimes happen within the same major version.

Behavior changes are more subtle and don't have common symptoms.

## Troubleshooting

### Diagnosing version mismatch issues

#### Dependency tree

Run `mvn dependency:tree` or `gradle dependencies — scan` to show full dependency tree with versions. (Note: `mvn dependency:tree -Dverbose` gives more information, but [may  be misleading](https://maven.apache.org/shared/maven-dependency-tree/)). When you suspect version conflict for some library, notice if multiple versions of it are detected and which components depend on it.

Note that dependency resolution in development and production environments may work differently. Here are some known environments with custom dependency resolution and/or need extra configuration for user-defined dependencies. Please check out corresponding project dependency management documentation.

- [Apache Spark](https://spark.apache.org/docs/latest/submitting-applications.html#bundling-your-applications-dependencies)
- [Apache Flink](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/dev/datastream/project-configuration/)
- [Databricks](https://kb.databricks.com/libraries/maven-library-version-mgmt.html)
- IDE plugins

#### Jackson runtime version detection

Since Azure Core 1.20.0-beta.1 we added runtime detection of Jackson version.

- In case of `LinkageError` (any of its subclasses) exception related to Jackson API, check message of the exception for runtime version information.</br>*Example*: `com.azure.core.implementation.jackson.JacksonVersionMismatchError: com/fasterxml/jackson/databind/cfg/MapperBuilder Package versions: jackson-annotations=2.9.0, jackson-core=2.9.0, jackson-databind=2.9.0, jackson-dataformat-xml=2.9.0, jackson-datatype-jsr310=2.9.0, azure-core=1.20.0-beta.1`

- Look for warning/error [logs](https://docs.microsoft.com/azure/developer/java/sdk/logging-overview) from `JacksonVersion`.</br>*Example*: `[main] ERROR com.azure.core.implementation.jackson.JacksonVersion - Version '2.9.0' of package 'jackson-core' is not supported (too old), please upgrade.`

### Mitigation

#### Using Azure SDK BOM

Please use latest stable [Azure SDK BOM - TODO new docs link](https://devblogs.microsoft.com/azure-sdk/dependency-management-for-java/#azure-sdk-bom) file and don't specify version for any Azure SDKs or their dependencies. When applicable, make sure you're also using [Azure Spring Boot BOM](https://mvnrepository.com/artifact/com.microsoft.azure/azure-spring-boot-bom). Using Azure BOMs avoids version conflicts within Azure ecosystem and between Azure ecosystem and application.

#### Adjusting library versions

If issue persist after switching to latest BOM, identify libraries causing conflict (see [Dependency tree](#dependency-tree)) and try to update their version to avoid conflict. If a library, your application depend upon, uses old version of Jackson, Netty or Reactor (or any other common dependency), please post an issue on that library repository and request update dependency version.
Avoid downgrading Azure SDK version to match dependency versions - this may expose your application to known vulnerability issues and other bugs.

#### Shading

In some cases, there is no reasonable combination of libraries that can work together or application may depend on specific version, while changing its code is not possible.
One way of resolving this issue is to introduce a 'shaded JAR' - a single jar file including the library code, as well as the code of all of its dependencies, with the dependency package names renamed into a different namespace to avoid conflicts. Note that this approach only resolves conflicts and does not address known vulnerabilities issues.

Create wrapper package applicable to your scenario:

1. **Transitive dependency**: E.g. third-party library `A` uses Jackson 2.9.X and it's not possible to update `A`. Create a new JAR which includes `A` and shades Jackson 2.9.X (you may shade other dependencies and `A` too if needed).
2. **Application dependency**: You application uses Jackson 2.9.X directly and while you're working on upgrading your code to the new version, you can shade Jackson 2.9.X by creating a new package with all Jackson libraries - check out the example below.

**Note**: shading Jackson into application JAR would not resolve version conflict as it forces all dependencies to use single shaded version of Jackson.

Example of shading Jackson libraries under a new JAR with Maven:

- Use [Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/).
- Create a new package that would wrap or Jackson libraries themselves if your application needs older version
- Configure shading plugin:

```xml
<plugin>
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-shade-plugin</artifactId>
<version>${maven-shade-plugin-version}</version>
<executions>
    <execution>
        <phase>package</phase>
        <goals>
            <goal>shade</goal>
        </goals>
        <configuration>
            <!--Create shaded JAR only-->
            <shadedArtifactAttached>false</shadedArtifactAttached>
            <!--Remove original replaced dependencies-->
            <createDependencyReducedPom>true</createDependencyReducedPom>
            <!--Promotes transitive dependencies of removed dependencies to direct-->
            <promoteTransitiveDependencies>true</promoteTransitiveDependencies>
            <artifactSet>
                <includes>
                    <include>com.fasterxml.jackson:*</include>
                    <include>com.fasterxml.jackson.*:*</include>
                </includes>
            </artifactSet>
            <relocations>
                <relocation>
                    <pattern>com.fasterxml.jackson</pattern>
                    <shadedPattern>org.example.shaded.com.fasterxml.jackson</shadedPattern>
                    <includes>
                        <include>com.fasterxml.jackson.**</include>
                    </includes>
                </relocation>
            </relocations>
        </configuration>
    </execution>
</executions>
</plugin>
```

- Run `mvn package` to create a Jackson wrapper JAR file: it does not depend on original Jackson packages, instead it includes renamed Jackson packages and classes. Make sure to update namespaces in your application code to `org.example.shaded.com.fasterxml.jackson.*` (or other prefix of your choice).

#### Azure Functions (Java 8) configuration

On Azure Functions (running Java 8 only) internal dependencies version take precedence over custom JARs. It may cause version conflicts especially with Jackson, Netty and Reactor.

**Solution**: set `FUNCTIONS_WORKER_JAVA_LOAD_APP_LIBS` environment variable to `true` or `1`. Make sure to update Azure Function Tools (v2 or v3) to latest version.

## Compatible versions

Please refer to [Maven Central](https://mvnrepository.com/artifact/com.azure/azure-core) for details on `azure-core` specific dependencies and their versions. Here are some general considerations:

- Jackson: 2.10 or newer (minor) versions are compatible.
- slf4j: v1.7.*.
- Netty:
  - io.netty:netty-tcnative-boringssl-static - 2.0.*
  - io.netty:netty-common, etc - 4.1.*
- Reactor: 3.X. *Major and minor* versions have to match exactly ones `azure-core` depends upon - refer to [Project Reactor breaking change policy](https://github.com/reactor/.github/blob/main/SUPPORT.adoc#our-policy-on-deprecations)

TODO: compatibility between Azure Spring Boot BOM and Azure SDK BOM
