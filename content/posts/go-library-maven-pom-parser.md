+++
title = "Go library to parse Maven POM file"
date = "2019-10-16"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["Golang"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

I have recently written a little go tool that parse maven pom file to analyse dependencies between two projects in order to perform detailed analysis such as evolution of project dependencies, etc...

After a bit of research I couldn't find any existing parser for Go and therefore I have decided to write one. As XML parsing is supported natively in Go there is not much work to do: only declare the structure that the Go XML parser will use.

Since my parser was working great I have decided to open source it: [github.com/creekorful/mvnparser](https://github.com/creekorful/mvnparser)

Let's take the following POM as example:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>javax.enterprise</groupId>
            <artifactId>cdi-api</artifactId>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.0</version>
                <configuration>
                    <release>11</release>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

You can parse it using the following code:

```go
package main

import (
	"github.com/creekorful/mvnparser"
	"encoding/xml"
	"log"
)

func main() { 
    // filled with previously declared xml 
    pomStr := "..."
	
    // Load project from string
    var project mvnparser.MavenProject
    if err := xml.Unmarshal([]byte(pomStr), &project); err != nil {
        log.Fatalf("unable to unmarshal pom file. Reason: %s", err)
    }
    
    log.Print(project.GroupId) // -> com.example
    log.Print(project.ArtifactId) // -> my-app
    log.Print(project.Version) // -> 1.0.0-SNAPSHOT
    
    // iterate over dependencies
    for _, dep := range project.Dependencies {
    	log.Print(dep.GroupId)
    	log.Print(dep.ArtifactId)
    	log.Print(dep.Version)
    	
    	// ...
    }
}
```

Happy hacking !