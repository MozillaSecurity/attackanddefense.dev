---
layout: post
title: "Firefox CodeQL Databases Available for Download"
date: 2020-05-25
categories: 
  - "bug-bounty"
---

In November of 2019 we added [static analysis bounties for CodeQL queries and Clang plugins](https://blog.mozilla.org/attack-and-defense/2019/11/14/adding-codeql-and-clang-to-our-bug-bounty-program/). Github has a [great CodeQL portal](https://securitylab.github.com/tools/codeql) with detailed instructions for [creating a database](https://help.semmle.com/codeql/codeql-cli/procedures/create-codeql-database.html) that will work efficiently with Firefox.

Generating that database might be a time consuming task, because building historical versions of Firefox can be bulky due to toolchain changes over time. On the flip side, building historical CodeQL databases of Firefox can be rewarding because [we will pay a bounty for CodeQL queries that match previously fixed security issues](https://www.mozilla.org/en-US/security/client-bug-bounty/#static-analysis-bounty).

Given all this, we are making CodeQL databases available for download for Firefox versions 68 through the present. You can find them on S3 at [https://bug-bounty-codeql-databases.s3.us-east-2.amazonaws.com/index.html](https://bug-bounty-codeql-databases.s3.us-east-2.amazonaws.com/index.html) - if you have any issues with the databases or find some missing, please email [security@mozilla.org](mailto:security@mozilla.org)

**Update** (October, 2022): Due to low usage and maintenance burden, we have stopped publishing codeql databases. However, you can still generate them locally. Our script for generating the JavaScript database [was here](https://searchfox.org/mozilla-central/rev/b4150d1c6fae0c51c522df2d2c939cf5ad331d4c/taskcluster/scripts/misc/generate-codeql-db-javascript.sh). The [the cpp database script](https://searchfox.org/mozilla-central/rev/b4150d1c6fae0c51c522df2d2c939cf5ad331d4c/taskcluster/scripts/builder/build-linux.sh) was more complicated and tied into the build mechanics, but the general approach of giving it a high memory limit and running `./mach build` will probably work.
