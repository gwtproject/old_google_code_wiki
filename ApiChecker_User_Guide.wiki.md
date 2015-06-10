# Introduction

Api Checker checks for any breakages in GWT's public api. Whenever you try to submit anything to GWT's code base, the api checker is run.

# The Api Checker Configuration

The Api Checker checks the current GWT code against a reference code. Typically, the reference code is the previous release, specified as the gwt-user.jar and gwt-dev.jar from the release. Since Api checker currently uses TypeOracle and TypeOracle only processed .java files when Api Checker was written, any .class files in the jars are not used by the Api Checker. So to reduce the  size of the jars, the .class files in the jars are removed (the createApiCheckerReferenceJars.sh script referred below does this). In addition to the reference code, the Api Checker requires a configuration file. The configuration file is organized as a properties file that specifies the following properties:
  * name\_old: specifies the name of the old or reference api.
  * sourceFiles\_old: specifies the root of the filesystem trees to be checked for building the api. This is a colon separated list.
  * excludedFiles\_old: specifies lists of ant patterns to exclude, i.e., the contents of these files will not be present in the api. Also, if a file contains code that cannot be parsed by GWT, such files need to be excluded here.

  * dirRoot\_new: specifies the base directory for all other file/directory names.
  * name\_new: same as above, but for the new api, i.e., the api being checked.
  * sourceFiles\_new: analogous to sourceFiles\_old
  * excludedFiles\_new: analogous to excludedFiles\_old

  * excludedPackages: list of colon-separated packages where api breakages are not to be flagged. If a package is listed here, the public api for any code in the package can be freely refactored.  One common use is to enforce GWT's convention regarding .impl packages. Any GWT code in `*.impl.*` package is not considered to be public api.
  * Api whitelist: the last part of the configuration file. It is a list of api changes, each change on a separate line, that have been white-listed. When adding an entry here, include comments as to why the entry is been added.


http://code.google.com/p/google-web-toolkit/source/browse/releases/2.1/tools/api-checker/config/gwt20_21userApi.conf shows an example config file.

The code for api checker is in tools/api-checker in the GWT trunk dir.  The different sub-dirs are:
  * config: contains the configuration files, as mentioned above.
  * reference: contains the reference jars.
  * src: contains the src. ApiCompatibilityChecker is the main java class.
  * test: contains the test code.


# What to do when the Api Checker fails?
Multiple possibilities here:
  * Failing to build type oracle: This might occur if you just added some server code that uses reflection or wrote a new generator. Since Api Checker cannot process such files, the only way out is to add an entry to excludeFiles\_new excluding such files.
  * Showing errors on impl package: To exclude the impl package from being processed, add it to the excludedPackages list.
  * Other api break: Look for the version of the class file in git/svn to see if it is a genuine API breakage. If you intend to whitelist the change, just copy the error message line and paste it at the end of the config file. Do add a comment indicating the reason for the whitelist entry.

To run api checker, type 'ant apicheck-nobuild' from your trunk dir.


# Bumping Api Checker after a release
(example below assumes gwt 2.1 has been released, and the next release is gwt 2.2)

The steps involved in the process are:
  * Update the jar files in tools/api-checker/reference -- Download the distribution zip, run createApiCheckerReferenceJars.sh
  * Create a new config file gwt21\_22userApi.conf.
    * Copy the gwt20\_21userApi.conf to gwt21\_22userApi.conf.
    * Update the name\_old and name\_new (for example, to gwt21userApi and gwt22userApi respectively).
    * copy excludedFiles\_new to excludedFiles\_old, remove the user/src prefix from each entry, add entries in the gwt20\_21userApi.conf/excludedFiles\_old that related to com/google/gwt/dev, com/google/gwt/core/ext/ and com/google/gwt/soyc/.
      * Getting this list right might take 1-2 iterations of running the tool, and putting in the appropriate excludes. (Anything under dev/core/src is not checked and needs to be put in this list.)
    * Remove everything from excludedPackages since there are no api changes.
    * Similarly, remove everything from the whitelist since there should not be any Api changes.
  * Modify the build.xml in the trunk dir to point to gwt21\_22userApi.conf instead of gwt20\_21userApi.conf