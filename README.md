sbt-docker
==========
sbt-docker is a [sbt](http://www.scala-sbt.org/) plugin which builds [Docker](http://www.docker.io/) images for your projects.

Requirements
------------
* sbt
* Docker

Setup
-----

Add sbt-docker as a dependency in `project/plugins.sbt`:
```scala
addSbtPlugin("se.marcuslonnberg" % "sbt-docker" % "1.2.0")
```

sbt-docker is an [auto plugin](http://www.scala-sbt.org/0.13/docs/Plugins.html),
this means that sbt version 0.13.5 or higher is required.

Java 7 or higher is required. Use version `1.1.0` if you need Java 6 support.

### Getting started

Below are some documentation on the sbt tasks and settings in the plugin.

This blog post gives a good introduction to the basics of sbt-docker: [Dockerizing your Scala apps with sbt-docker](https://velvia.github.io/Docker-Scala-Sbt/)

Also take a look at the [example projects](examples).

Usage
-----

Start by enabling the plugin in your `build.sbt` file:
```scala
enablePlugins(DockerPlugin)
```

This sets up some settings with default values and adds tasks such as `docker` which builds a Docker image.
The only required setting that is left to define is `dockerfile in docker`.

### Artifacts

If you want your Dockerfile to contain one or several artifacts (such as JAR files) that your
project generates, then you must make the `docker` task depend on the tasks that generate them.
It could for example be with the `package` task or with tasks from plugins such as
[sbt-assembly](https://github.com/sbt/sbt-assembly).

Here is how to make the docker task depend on the sbt `package` task:
```scala
docker <<= docker.dependsOn(`package`.in(Compile, packageBin))
```

### Defining a Dockerfile

In order to produce a Docker image a Dockerfile must be defined.
It should be defined at the `dockerfile in docker` key.
There is a mutable and an immutable Dockerfile class available, both provides a DSL which resembles
the plain text [Dockerfile](https://docs.docker.com/reference/builder/) format.
The mutable class is default and is used in the following examples.

Example with the sbt `package` task.
```scala
docker <<= docker.dependsOn(`package`.in(Compile, packageBin))

dockerfile in docker := {
  val jarFile = artifactPath.in(Compile, packageBin).value
  val classpath = (managedClasspath in Compile).value
  val mainclass = mainClass.in(Compile, packageBin).value.getOrElse(sys.error("Expected exactly one main class"))
  val jarTarget = s"/app/${jarFile.getName}"
  // Make a colon separated classpath with the JAR file
  val classpathString = classpath.files.map("/app/" + _.getName)
    .mkString(":") + ":" + jarTarget
  new Dockerfile {
    // Base image
    from("java")
    // Add all files on the classpath
    add(classpath.files, "/app/")
    // Add the JAR file
    add(jarFile, jarTarget)
    // On launch run Java with the classpath and the main class
    entryPoint("java", "-cp", classpathString, mainclass)
  }
}
```

Example with the [sbt-assembly](https://github.com/sbt/sbt-assembly) plugin:
```scala
// Make the docker task depend on the assembly task, which generates a fat JAR file
docker <<= (docker dependsOn assembly)

dockerfile in docker := {
  val artifact = (outputPath in assembly).value
  val artifactTargetPath = s"/app/${artifact.name}"
  new Dockerfile {
    from("java")
    add(artifact, artifactTargetPath)
    entryPoint("java", "-jar", artifactTargetPath)
  }
}
```

Have a look at [DockerfileExamples](examples/DockerfileExamples.scala) for different ways of defining a Dockerfile.

### Building an image

To build an image use the `docker` task.
Simply run `sbt docker` from your prompt or `docker` in the sbt console.

### Pushing an image

An image that have already been built can be pushed with the `dockerPush` task.
To both build and push an image use the `dockerBuildAndPush` task.

The `imageNames in docker` key is used to determine which image names to push.

### Custom image names

You can specify the names / tags you want your image to get after a successful build with the `imageNames in docker` key of type `Seq[sbtdocker.ImageName]`.

Example:
```scala
imageNames in docker := Seq(
  ImageName(s"${organization.value}/${name.value}:latest"), // Sets the latest tag
  ImageName(
  	namespace = Some(organization.value),
    repository = name.value,
    tag = Some("v" + version.value)
  ) // Sets a name with a tag that contains the project version
)
```

### Build options

Use the key `buildOptions in docker` to set build options.

Example:
```scala
buildOptions in docker := BuildOptions(
  cache = false,
  removeIntermediateContainers = BuildOptions.Remove.Always,
  pullBaseImage = BuildOptions.Pull.Always)
```

### Auto packaging JVM applications

If you have a standalone JVM application that you want a simple Docker image for.
Then you can use `dockerAutoPackageJavaApplication(fromImage, exposedPorts, exposedVolumes, username)`
which will setup some settings for you, including a Dockerfile.
Its very basic, so if you have more advanced needs then define your own Dockerfile.
