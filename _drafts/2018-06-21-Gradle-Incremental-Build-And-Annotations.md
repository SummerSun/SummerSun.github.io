---
layout: post
title:  "Gradle Incremental Build And Annotations"
categories: ONE
---

Learning Gradle.

@Input and @OutputFile/@OutputDirectory could increase build, or, define incremental build.

Gradle create custom tasks, define all inputs, outputs. No structure, so Extension Concepts are introduced.

But @Input and @Output* are not applied to Extension, so incremental build seems are dropped during the development.
