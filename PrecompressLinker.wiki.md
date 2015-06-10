# Introduction

We generally recommend that the static content of a web application be served compressed, because it reduces the latency of the network transfer.  To implement such compression, many projects compress the files ahead of time as part of their build.  We could support this by adding precompression as an optional linker in the SDK.

To contrast, without such a linker, people need to implement the compression as an after-the-fact build step.  For a parallel build this is much slower, because all the compression happens sequentially on one computer.

# Goals

Help developers do the right thing.  Currently developers must implement precompression themselves.  We can make this something they simply turn on in their module file.

Make builds faster.  A linker can be sharded and can update the content before it makes it into a jar at all.

# Non-goals

Support server configuration to make use of precompressed content.

# Details

We define a linker "precompress".  People who want it add the following line to their gwt.xml file:
```
<inherits name="com.google.gwt.precompress.Precompress"/>
```

It's not added by default because not all web servers will support precompressed resources.  However, most web servers should be possible to configure such that when a request comes for foo.html, it checks for the existence of foo.html.gz and uses that if present and the client said it accepted compression.

They would additionally need to specify what files they want to be precompressed.  A config property would allow adding and removing patterns of files to be compressed, just like with the RPC whitelist:
```
<!-- add a regex (a + sign is optional) -->
<extend-configuration-property name='precompress.path.regexes' value='.*\.xml' />

<!-- remove a regex -->
<extend-configuration-property name='precompress.path.regexes' value='-foo\.css' />
```

The regex will be matched against the "partial path" of the artifact, i.e. its path relative to the top level of the emitted public artifacts.  If it starts with a plus sign, it means the file should be compressed.  With a minus sign, don't compress it.  The last matching pattern in the list is the one that counts.

The default patterns are as follows:
  * `.*\.html`
  * `.*\.js`
  * `.*\.css`
For compatibility with Jetty's DefaultServlet, among with other configurations, the original files are by default left in the output.  However, the following setting causes the original files to be removed, leaving only the .gz files.
```
  <set-configuration-property name='precompress.leave.originals' value='false' />
```