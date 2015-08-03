---
layout: post
title: Eclipse使用Maven创建Web时报错：Could not resolve archetype
categories: Java
tags: Java Eclipse Maven
---

问题描述：

	使用Eclipse自带的Maven插件创建Web项目时报错：

		Could not resolve archetype org.apache.maven.archetypes:maven-archetype-webapp:RELEASE from any of the configured repositories.
		Could not resolve artifact org.apache.maven.archetypes:maven-archetype-webapp:pom:RELEASE
		Failed to resolve version for org.apache.maven.archetypes:maven-archetype-webapp:pom:RELEASE: Could not find metadata org.apache.maven.archetypes:maven-archetype-webapp/maven-metadata.xml in local (C:\Users\liujunguang\.m2\repository)
		Failed to resolve version for org.apache.maven.archetypes:maven-archetype-webapp:pom:RELEASE: Could not find metadata org.apache.maven.archetypes:maven-archetype-webapp/maven-metadata.xml in local (C:\Users\liujunguang\.m2\repository)

解决方法：

	脑子一抽把.m2文件夹删除了，再新建就不报错了，暂时还没研究原理。