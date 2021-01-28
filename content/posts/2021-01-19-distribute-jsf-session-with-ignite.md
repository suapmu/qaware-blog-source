---
title: "Distribute Jsf Sessions With Ignite"
date: 2021-01-19
draft: true
author: "Susanne Apel"
tags: ["Ignite", "JSF", "Cloud", "Migration"]
---

## Motivation

Within a resent project, we were working in a softwar migration project
insurance industry: applications formally running on an IBM Portal server
as JSF portlets
now were targeted to run standalone in a enterprise cloud environment.

This article is about the session handling of these applications.
In the old world, the sessions were sticky sessions, i.e. assosiated with
a running instance of the portal server. This means that every restart of
a portal server would result in a loss of session data. This was ok
in the old world since restarts and deployments were rare and mostly
management decisions. Running an application in a cloud environment,
rolling redeployments at least of un-changed software should not cause problems.
Otherwise, it would conflict with all resilience mechanisms a cloud comes with.

Since, in an earlier, similar migration project, we made good
experiences with ignite
<!-- TODO: reference-->
the plan was to use ignite again for session distriubtion.

The present post describes the special challenges
which were found in the migrated JSF applications and
 why they were solved how.

<!-- todo: behalten? -->
In conclusion, there are a few more comments for future operations
or further future adjustments.


