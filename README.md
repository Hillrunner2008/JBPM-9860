# Steps to reproduce https://issues.redhat.com/browse/JBPM-9860

## Step 1 (build the project and pay attention to the output from the maven-assembly-plugin)
```bash
$ mvn clean install;
$ cd app/target/local-repository/maven/

# Notice the maven directory tree content contains only guava-29.0-jre.jar (as it should)
guava
│
├── 29.0-jre
```


## Step 2 (Configure a custom maven repository containing only the content above)

### Example wildfly standalone.xml configured system property to achieve this

```xml
<system-properties>
        <property name="kie.maven.settings.custom" value="/path/to/custom-settings.xml"/>
        ...
</system-properties>
```

where custom-settings.xml contains the full path to the maven-assembly directory in its "localRepository" tag.


## Step 3 Deploy the kjar (while offline)
If you deploy while online you can only confirm the conflicted dependencies were resolved into your maven repository directory. Deploying while offline is important for illustrating that deployment failure will result.

```bash
curl --location --request PUT 'http://localhost:8080/kie-server/services/rest/server/containers/mybusinessapp' \
-H 'Content-Type: application/json' \
--data-raw '{
    "container-id" : "mybusinessapp",
    "release-id" : {
        "group-id" : "org.kie.businessapp",
        "artifact-id" : "mybusinessapp",
        "version" : "1.0"
    }
}'
```
## Review the logs 
```
Caused by: org.eclipse.aether.collection.DependencyCollectionException: Failed to collect dependencies at org.kie.businessapp:model:jar:1.0 -> com.google.guava:guava:jar:30.1.1-jre
	at deployment.kie-server.war//org.eclipse.aether.internal.impl.DefaultDependencyCollector.collectDependencies(DefaultDependencyCollector.java:291)
	at deployment.kie-server.war//org.eclipse.aether.internal.impl.DefaultRepositorySystem.collectDependencies(DefaultRepositorySystem.java:316)
	at deployment.kie-server.war//org.appformer.maven.integration.MavenRepository.getArtifactDependecies(MavenRepository.java:126)
	... 76 more
Caused by: org.eclipse.aether.resolution.ArtifactDescriptorException: Failed to read artifact descriptor for com.google.guava:guava:jar:30.1.1-jre
	at deployment.kie-server.war//org.apache.maven.repository.internal.DefaultArtifactDescriptorReader.loadPom(DefaultArtifactDescriptorReader.java:282)
	at deployment.kie-server.war//org.apache.maven.repository.internal.DefaultArtifactDescriptorReader.readArtifactDescriptor(DefaultArtifactDescriptorReader.java:198)
	at deployment.kie-server.war//org.eclipse.aether.internal.impl.DefaultDependencyCollector.resolveCachedArtifactDescriptor(DefaultDependencyCollector.java:535)
	at deployment.kie-server.war//org.eclipse.aether.internal.impl.DefaultDependencyCollector.getArtifactDescriptorResult(DefaultDependencyCollector.java:519)
	at deployment.kie-server.war//org.eclipse.aether.internal.impl.DefaultDependencyCollector.processDependency(DefaultDependencyCollector.java:409)
	at deployment.kie-server.war//org.eclipse.aether.internal.impl.DefaultDependencyCollector.processDependency(DefaultDependencyCollector.java:363)
	at deployment.kie-server.war//org.eclipse.aether.internal.impl.DefaultDependencyCollector.process(DefaultDependencyCollector.java:351)
	at deployment.kie-server.war//org.eclipse.aether.internal.impl.DefaultDependencyCollector.collectDependencies(DefaultDependencyCollector.java:254)
```


You can see the side effect of this https://github.com/kiegroup/kie-soup/blob/master/kie-soup-maven-utils/kie-soup-maven-integration/src/main/java/org/appformer/maven/integration/ArtifactResolver.java#L155 code is the above kjar deployment failure when in reality all necessary classpath content was in place in the maven repository. 

## Expected behavior 
Do not resolve dependencies with some custom code that doesn't match maven