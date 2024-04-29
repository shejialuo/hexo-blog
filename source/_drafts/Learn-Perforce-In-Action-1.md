---
title: Learn Perforce In Action (1)
tags:
  - tag_name
categories:
  - category_name
---

## Preface

The reason why I write these series is that I really want to understand the usage of Perforce. There are so many wonderful books published by manning called "xx in action". What is the real understanding? Well, do it and know it.

## Overview

We first need to understand the architecture of the Perforce. In this tutorial, we mainly compare Perforce with the Git. They are totally different. And I think most of the readers may have some experience with Git. If you haven't worked at a company, you may never use Perforce before.

Unlike Git, Perforce is a centralized version control software. When you are working with Perforce, you must have connection with the server. So you can guess the basic architecture of the Perforce. The Perforce is a classic client-server architecture. The server would store everything, you do not consider about the server except the access control. Because Perforce is a commercial software, we do not know how does it store the contents.

There are many clients you can interact with Perforce server. In this tutorial, we mainly talk about the command line client. Before we start, we need to first install the Perforce server and Perforce client.

## Installation

We could easily install the server and client from the [official website](https://www.perforce.com/downloads/helix-command-line-client-p4). Depend on your operating system, you should download different binary.

If you successfully download the binary, you will find the following two executable binary:

+ `p4`: the command line client.
+ `p4d`: the server.

## First Try

Now, we can start our first try for Perforce. We should first start the server and we need to specify the directory where we store thing. In Perforce terminology
