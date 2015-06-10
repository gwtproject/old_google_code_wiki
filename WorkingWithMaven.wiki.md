This page is intended to serve as the canonical reference for working with Maven in the latest GWT and GPE releases. New releases of 3rd-party projects like Eclipse don't necessarily correspond to GWT releases, hence the need for a place to publish new notes independent of a release.



# Maven GWT Plugin
In order to include GWT in a Maven POM, you'll want [Maven GWT Plugin](http://mojo.codehaus.org/gwt-maven-plugin/), a third-party project with its own community support.

# Sample Projects
The easiest way to get a GWT POM going is to snag one from of the GWT Maven sample projects below. There are currently issues with the gwt-maven-plugin archetypes, so a good starter POM is the best way to go until those issues are resolved.

The GWT team maintains two sample projects using Maven in GWT trunk. These are

  * [Mobile Web app sample](https://gwt.googlesource.com/gwt/+/master/samples/mobilewebapp/)
  * [DynaTable RequestFactory sample](https://gwt.googlesource.com/gwt/+/master/samples/dynatablerf/)


Several other Maven samples are available from the GWT community:

  * [listwidget (App Engine, Objectify)](http://code.google.com/p/listwidget/)
  * [gwt-sample (Spring, JPA2)](https://bitbucket.org/gardellajuanpablo/gwt-sample/wiki/Home)

# Using Maven with Google Plugin for Eclipse
<a />
[Google Plugin for Eclipse](https://developers.google.com/eclipse/) (GPE) makes it easy to import Maven projects using GWT and App Engine. To get started, first
[install GPE](https://developers.google.com/eclipse/docs/getting_started) and the additional Eclipse plugins required for your version of Eclipse (only Indigo+ is supported at this time):

## Eclipse Indigo/Juno Java IDE (**not** Java EE)
  1. Install m2e-wtp from update site `http://download.jboss.org/jbosstools/updates/m2eclipse-wtp`. Install the following features:
    * Maven Integration for WTP (required)
  1. If you're using RequestFactory, install m2e-apt from update site `http://download.jboss.org/jbosstools/updates/m2e-extensions/m2e-apt`. Install the following features:
    * Maven Integration for JDT APT

## Eclipse Indigo/Juno Java EE IDE
  1. Install m2e-wtp from update site `http://download.jboss.org/jbosstools/updates/m2eclipse-wtp`. The following features are required:
    * Maven Integration for Eclipse (required)
    * Maven Integration for WTP (required)
  1. If you're using RequestFactory, install m2e-apt from update site `http://download.jboss.org/jbosstools/updates/m2e-extensions/m2e-apt`. Install the following features:
    * Maven Integration for JDT APT

## Newer Eclipse releases
  1. Eclipse should come with Maven Integration for Eclipse preinstalled, but might not have Maven Integration for WTP. If needed, install it from the standard Eclipse update site for your release.
  1. If you're using RequestFactory, install m2e-apt from update site `http://download.jboss.org/jbosstools/updates/m2e-extensions/m2e-apt`. Install the following features:
    * Maven Integration for JDT APT

After installing GPE and the required plugins for your version of Eclipse, you're ready to work with a GWT Maven project.

## Configure a Maven project in Eclipse
To configure an existing GWT Maven project, simply choose File | Import | Existing Maven project in Eclipse and browse to the project's pom.xml file. Google Plugin for Eclipse will automatically configure the project source folders and the GWT and App Engine SDKs if present in the POM.

If you're using RequestFactory, you'll need to enable m2e's annotation processing functionality as well. Under project properties, select Maven > Annotation Processing > Enable Project-Specific Settings, and choose the "Automatically configure JDT APT. Click "Finish", and then right-click on the project, and **select Maven > Update project** (very important step).

Now you can run the project using Run as | Web application or by running Maven from the command line or the project's Maven menu.

## Troubleshooting
If you have problems importing a Maven project with Google Plugin for Eclipse, verify the following:
  * You have installed the correct Maven helper plugins above for your version of Eclipse
  * You have installed the most recent Google Plugin for Eclipse
  * You are using Eclipse 3.7 (Indigo) or higher
  * If needed, you've enabled Annotation Processing for m2e under project properties > Maven > Annotation Processing, and after doing this, you've updated the project configuration by right-clicking on the project and selecting Maven > Update Project.

If a project has been imported correctly, you should see the following:
  * Under project properties, Google > Web Toolkit, the "Use Google Web toolkit" box should be checked and "Use specific SDK" should point to GWT in your local Maven repository
  * Under project properties, Google > Web Application, the "This project has a WAR directory" box should be checked and "WAR directory" should be src/main/webapp
  * src folders should appear as Eclipse source folders in Package Explorer (except src/main/webapp, which will show as a normal folder)

If project import fails due to a POM error, fix the error and retry the import as follows:
  1. Remove the project in Eclipse (but don't delete source folders)
  1. At the command line in the directory containing the POM, run "mvn clean eclipse:clean"
  1. Try the import again

In most cases, after making a POM change in Eclipse, you should be able to right-click on the project, then click Maven > Update project configuration and/or Update project dependencies. However, the steps above are the surest way to pick up all changes.

# Advanced topics

## POM changes needed for Eclipse Indigo
Due to changes in the m2e Eclipse plugin for Indigo (3.7) and higher, plugin execution no longer happens directly, but rather must be configured through m2e's lifecycle-mapping plugin. In order for a plugin to be invoked within Eclipse, you must define a pluginExecution in the lifecycle-mapping plugin corresponding to the execution tag in the plugin definition itself. To achieve this, add a new section within the build section of the POM. The sample below shows the definitions needed to support maven-gae-plugin and maven-datanucleus-plugin (also for App Engine).

```
    <pluginManagement>
      <plugins>
        <!--This plugin's configuration is used to store Eclipse m2e settings only. It has no influence on the Maven build itself.-->
        <plugin>
          <groupId>org.eclipse.m2e</groupId>
          <artifactId>lifecycle-mapping</artifactId>
          <version>1.0.0</version>
          <configuration>
            <lifecycleMappingMetadata>
              <pluginExecutions>
                <pluginExecution>
                  <pluginExecutionFilter>
                    <groupId>org.datanucleus</groupId>
                    <artifactId>maven-datanucleus-plugin</artifactId>
                    <versionRange>[1.1.4,)</versionRange>
                    <goals>
                      <goal>enhance</goal>
                    </goals>
                  </pluginExecutionFilter>
                  <action>
                    <ignore></ignore>
                  </action>
                </pluginExecution>
                <pluginExecution>
                  <pluginExecutionFilter>
                    <groupId>net.kindleit</groupId>
                    <artifactId>maven-gae-plugin</artifactId>
                    <versionRange>[0.7.3,)</versionRange>
                    <goals>
                      <goal>unpack</goal>
                    </goals>
                  </pluginExecutionFilter>
                  <action>
                    <execute />
                  </action>
                </pluginExecution>
              </pluginExecutions>
            </lifecycleMappingMetadata>
          </configuration>
        </plugin>
      </plugins>
    </pluginManagement>
```

## Using a Maven multi-project with GWT

TODO