---
layout: post
title:  "Using the Wrong Version of a Dependency"
date:   2021-09-15 11:20:14 -0800
categories: development
---
This is the story of my first encounter debugging a problem with multiple versions of a dependency in a Maven project.

Using [cookiecutter], I had just generated a new project off of a template pre-configured with various frameworks and libraries. I then added `software.amazon.awssdk:dynamodb-enhanced` to the pom, which was able to build succesfully. However, running tests failed with the following exception:
{% highlight error %}
java.lang.BootstrapMethodError: call site initialization exception
    at java.lang.invoke.CallSite.makeSite(CallSite.java:341)
    at java.lang.invoke.MethodHandleNatives.linkCallSiteImpl(MethodHandleNatives.java:307)
    at java.lang.invoke.MethodHandleNatives.linkCallSite(MethodHandleNatives.java:297)
    at software.amazon.awssdk.http.apache.ApacheHttpClient.transformHeaders(ApacheHttpClient.java:234)
    at software.amazon.awssdk.http.apache.ApacheHttpClient.createResponse(ApacheHttpClient.java:223)
    ...
{% endhighlight %}
followed by:
{% highlight error %}
Caused by: java.lang.invoke.LambdaConversionException: Invalid receiver type interface
org.apache.http.Header; not a subtype of implementation type interface
org.apache.http.NameValuePair
{% endhighlight %}

There was a mismatch between the `org.apache.http.Header` that the SDK was using and the one that was in the classpath of the project. It turned out [the issue][github-issue-652] had been reported, and [a change][httpcore-release-notes] in `org.apache.httpcomponents:httpcore:4.4.9` was made to "Make interface Header extend NameValuePair". `mvn dependency:tree -Dincludes=org.apache.httpcomponents:httpcore` showed that another library had declared `httpcore:4.4.4` as a dependency, which was being used for the project. 

So, I explicitly declared the latest version of `org.apache.httpcomponents:httpcore` in `<dependencyManagement>` and left a comment to remove it when the library updates its dependency on `httpcore` to `4.4.9` or up. Below is a snippet of the dependency declaration in the pom:
{% highlight xml %}
<properties>
  ...
  <httpcore.version>4.4.13</httpcore.version>
</properties>
...
<dependencyManagement>
  <dependencies>
    ...
    <dependency>
      <groupId>org.apache.httpcomponents</groupId>
      <artifactId>httpcore</artifactId>
      <version>${httpcore.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>
{% endhighlight %}

This fixed my issue and made sure I wasn't using conflicting versions of `httpcore`.

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
[cookiecutter]: https://github.com/cookiecutter/cookiecutter
[github-issue-652]: https://github.com/aws/aws-sdk-java-v2/issues/652
[httpcore-release-notes]: https://downloads.apache.org/httpcomponents/httpcore/RELEASE_NOTES-4.4.x.txt