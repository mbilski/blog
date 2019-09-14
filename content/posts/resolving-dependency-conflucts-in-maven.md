---
title: Resolving dependency conflicts in maven
date: 2015-03-30 21:15:50
---

Sometimes it is necessary to use the same library in two different versions in one application. This is when dependency  hell arises. This post describes how to quickly resolve it in maven.

<!--more-->

## Problem

The problem is typical and well known. An application has two maven modules *M1* and *M2*. *M1* depends on a library *L1*, *M2* depends on a library *L2*. *L1* and *L2* depend on a library *L3*. However, *L1* requires version 1.0, while *L2* requires version 2.0. Those versions are incompatible.

Although, it might be possible to compile such an application, the *IllegalArgumentException*, *ClassNotFoundException*, etc exceptions will be thrown during the runtime.

For *OSGi* that problem can be easily solved using bundle encapsulation. However, developers of standalone or container based java apps need to deal with a single JVM runtime where dependency conflicts are inevitable.

## Solution

So, how to resolve such dependency conflicts? The [Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/examples/class-relocation.html) comes to the rescue. It enables to move classes from one package to another by rewriting the bytecode.

![](/img/shading.png)

Let's say library *L3*'s classes are under *q.w.e* package. Modules *M1* and *M2* requires different versions of *L3*. In *M2* we want to move *L3*'s classes from *q.w.e* to *s.q.w.e* package. That way both versions will be loaded during runtime. To do that we need to add following plugin to the *pom.xml* of *M2* module.

``` xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-shade-plugin</artifactId>
      <version>2.3</version>
      <executions>
        <execution>
          <phase>package</phase>
          <goals>
            <goal>shade</goal>
          </goals>
          <configuration>
            <relocations>
              <relocation>
                <pattern>q.w.e</pattern>
                <shadedPattern>s.q.w.e</shadedPattern>
              </relocation>
            </relocations>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

And that's it! Everything in *M2* will use *s.q.w.e* package to call *L3* classes now. The *M2* jar file is now an *uber jar* which consists of all its dependencies which have overwritten bytecode.

{% blockquote %}
There might be a problem while using CDI (or other dependency injection framework) in this solution. I needed to exclude the shaded packages from scanning using following *WEB-INF/beans.xml*
{% endblockquote %}

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://xmlns.jcp.org/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:weld="http://jboss.org/schema/weld/beans"
       xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee/beans_1_1.xsd"
       version="1.1" bean-discovery-mode="all">
  <weld:scan>
    <weld:exclude name="s.q.w.e"/>
  </weld:scan>
</beans>
```

## Summary

Maven Shade plugin relocations feature offers easy and quick way to resolve dependency conflicts in maven projects. The main disadvantage of this solution is rapidly growing size of jar files, as all the bytecode needs to be translated. However, it can be a good workaround while waiting for the new version of a library with updated dependencies.
