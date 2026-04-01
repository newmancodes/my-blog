+++
date = '2026-03-07T18:00:00Z'
draft = true
title = 'Leveraging Linux Containers on Github Actions Windows Runners'
tags = ['github actions', '.net', 'containerisation']
ShowToc = true
TocOpen = true
+++

## Introduction

Recently at work we had a desire to support some CI/CD based automated testability for a codebase which made use of some windows-specific APIs. We make use of GitHub Actions for our automation needs I wanted to see how we could apply modern software engineering principles such as making use of containerisation technologies. A wrinkle I discovered very quickly was that GitHub Actions Runners built on the Windows Operating System support Windows containers, not Linux containers. This appears to be a bone of contention in the community:

- [Windows Runner with Linux Service Container #187246](https://github.com/orgs/community/discussions/187246)
- [Github actions - Is it possible to pull a Linux based image using Linux containers in Docker on a Windows runner?](https://stackoverflow.com/questions/78424687/github-actions-is-it-possible-to-pull-a-linux-based-image-using-linux-containe)

This blog post will walk you through the attempts I made, what I encountered, and how I arrived at an acceptable solution. It provides examples that you can adapt for your needs, the examples cover:

- Docker Compose
- Testcontainers
- Aspire

## Uncovering the Problem

## Finding a Solution

### Docker Compose

### Testcontainers

### Aspire

## Summary


