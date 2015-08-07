---
layout: post
title: hibernate3.jar和hibernate-core.jar的区别
categories: Java Hibernate
tags: Java Eclipse Maven Hibernate
---

今天按照教程做一个SSH的例子，向项目中添加hibernate支持，教程中没有使用maven，遂自己上网到maven库中查询hibernate的jar包，教程中使用的是hibernate3.jar，maven库中并没有这个jar包，倒是hibernate-core之类的一大堆，这可糊涂了，这两个jar包有什么区别呢？

上网搜索一番也没找到具体的解释，打开hibernate3.jar，发现hibernate-distribution-3.6.0.Final.pom，噢，原来这个jar包应该叫hibernate-distribution啊，那么和core包又有啥区别呢？

在pom中发现如下内容：

        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-core</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-ehcache</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-jbosscache</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-infinispan</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-oscache</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-swarmcache</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-c3p0</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-proxool</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-entitymanager</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>hibernate-envers</artifactId>
            <version>${project.version}</version>
        </dependency>

因此，初步推测，hibernate3.jar相当于hibernate-core.jar，hibernate-ehcache.jar等的组合。