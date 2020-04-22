
# Changes for JDK 11

To run Gradle in the IntelliJ IDE the following must be done:

* Use the right button on the top project node
  * `Open Module settings`
    * Set the JDK to version 11
  * In the same dialog box
    * Make sure the `Libraries` are all for JDK11
    * Make sure the `Project` JVM is set to JDK11
  * In the Gradle tab on the right
    * Select Gradle settings
      * Make sure the `Gradle JVM` is set to JDK 11

At this point you can use the IDE's Gradle-tool to run and execute the 
commands. However, it is not possible to:

* Execute the Gradle commands in the command line
* Execute the resulting application in the `ìmage2latex-mathpix/build/install/bin` 
directory. We get the following error: 

```
* What went wrong:
java.lang.UnsupportedClassVersionError: org/openjfx/gradle/JavaFXPlugin has been compiled by a more recent version of the Java Runtime (class file version 55.0), this version of the Java Runtime only recognizes class file versions up to 52.0
> org/openjfx/gradle/JavaFXPlugin has been compiled by a more recent version of the Java Runtime (class file version 55.0), this version of the Java Runtime only recognizes class file versions up to 52.0

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org
```

The problem here is that [javafx-gradle-plugin](https://github.com/openjfx/javafx-gradle-plugin),
which is used to identify and download the javafx binaries (depending on the 
OS), was compiled for a more recent JDK. The above command will only work if 
the JDK is a version compatible with the minimum Gradle requirements. This is 
because the Gradle plugin's JDK version is greater that the OS JDK. If we had 
a JDK greater that the plugin's JDK version, we would be assured binary 
compatibility. 
    
Before setting up Gradle to use the correct lets see how we can determine 
were the OS has placed the JDK. The for command lists the JDKs that are 
available in the system:

```
user@machine:~$ update-alternatives --list java
/usr/lib/jvm/java-11-openjdk-amd64/bin/java
/usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
/usr/lib/jvm/java-8-oracle/jre/bin/java
```  

The following gets us the current active path:

```
user@machine:~$readlink -f /usr/bin/java
/usr/lib/jvm/java-11-openjdk-amd64/bin/java
``` 

The issue here is why is the plugin JDK version higher than the OS JDK? 
Shouldn't Gradle downlaod a compatible version according to the OS JDK?
Strangely enough the IDE sets the JDK explicitly, sio we will also do that.
We have the following options:

Using the command line directly:    
```bash
./gradlew -Dorg.gradle.java.home=/path_to_jdk_directory
```

In the `gradle.properties` file in the project root set:
```
org.gradle.java.home=/path_to_jdk_directory
```

In your `build.gradle` file in the project root set:
```
compileJava.options.fork = true
compileJava.options.forkOptions.executable = /path_to_javac
```

If you use `gradle.properties` your build depends on that particular 
path. Instead, run Gradle tasks with following command line parameter:
```
gradle build -Dorg.gradle.java.home=/JDK_PATH
gradlew build -Dorg.gradle.java.home=/JDK_PATH
```

We opted for `gradle.properties` file and when running the application got:

```
* What went wrong:
Execution failed for task ':jre'.
> Directory not found: jdk/linux-jdk/jmods
```

The problem stems from the following Gradle Goovy snippet (target platform 
linux):  

```
runtime {
    imageZip = file("${buildDir}/../releases/Image2LaTeX-${version}.zip")
    addOptions("--strip-debug", "--compress", "2", "--no-header-files", "--no-man-pages")
    addModules("java.base", "java.datatransfer", "java.desktop", "java.logging", "java.net.http", "java.prefs", "java.sql", "java.transaction.xa", "jdk.unsupported", "jdk.unsupported.desktop", "java.xml")
    if (OperatingSystem.current().isLinux()) {
        targetPlatform("linux", "jdk/linux-jdk")
    } else if (OperatingSystem.current().isMacOsX()) {
        targetPlatform("macos", "jdk/osx-jdk.jdk/Contents/Home")
    } else if (OperatingSystem.current().isWindows()) {
        targetPlatform("windows", "jdk/windows-jdk")
    } else {
        throw new  GradleException("Unnown OS: ${osName}:${osName}")
    }
}
```

The error is that the JDK path that is set in the [Badass runtime plugin](https://github.com/beryx/badass-runtime-plugin)
`targetPlatform` is incorrect. The issue is that this project downloads and
generates the run-times for several OSs. We can see this in the CI/CD 
`.travis.yml` script. The corresponding jdk are explicitly downloaded and then
copied to the `jdk` directory using the `./scripts/jdk_setup.sh` file. This 
means that all of the scripts in `build` folder that expect this `jdk` path 
will fail (for example in the `build/install` directory). To circumvent this we
replace the line: 
  
    targetPlatform("linux", "jdk/linux-jdk")
with:

    import org.gradle.internal.jvm.Jvm
    targetPlatform("linux", Jvm.current().getJavaHome().getAbsolutePath())

and now we can execute the resulting image directly so:

```
cd ~/IdeaProjects/img2latex-mathpix/build/image/Image2LaTeX-linux/bin/Image2LaTeX
./Image2LaTeX
```

Note that irrespective the corrections above, the following also works: 

```
./gradlew -Plinux run
```

For more information on the above issues see [issue 74](https://github.com/blaisewang/img2latex-mathpix/issues/74).

Summary:

From the project root we can (1) start all over, (2) execute the app directly, 
(2) generate an archive and (3) dist, (3) generate an image and then (c) execute 
that image.  
       
```
./gradlew clean
./gradlew -Plinux run
./gradlew -Plinux runtimeZip
./gradlew -Plinux installDist
./gradlew -Plinux runtime
./build/image/Image2LaTeX-linux/bin/Image2LaTeX
```

In order to run the Gradle script successfully we need set the parameter `-Plinux` in
the script. Instructions to this are in the [Working with Gradle Tasks](https://www.jetbrains.com/help/idea/work-with-gradle-tasks.html)
help item. One must right click on the task to launch the dialog box that 
allows this. Unfortunately this does [not seem to be working](https://youtrack.jetbrains.com/issue/IDEA-202704)
and it also does not seem to be possible to [set this globally])(https://github.com/b1f6c1c4/gradle-run-with-arguments).
To use the IDE we can circumvent the flag by changing the build script lines:

    if (project.hasProperty("linux")) {
  
with   

    if (OperatingSystem.current().isLinux()) {

# Implementing the Capture Screen

https://gist.github.com/am4dr/b3b60ff78fd4264e0b840a7a6f134cf1
https://stackoverflow.com/questions/41287372/how-to-take-snapshot-of-selected-area-of-screen-in-javafx
http://java-buddy.blogspot.com/2016/01/javafx-example-to-capture-screengui.html
https://openjfx.io/javadoc/12/javafx.graphics/javafx/scene/robot/Robot.html
https://www.codejava.net/java-se/graphics/how-to-capture-screenshot-programmatically-in-java
https://blog.ngopal.com.np/2012/09/10/javafx-liveview/
