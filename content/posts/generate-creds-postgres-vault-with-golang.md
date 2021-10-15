---
title: "Generate Postgres credentials with Hashicorp Vault and Go"
date: "2021-10-15"
tags: [
    "go", "golang", "vault", "postgres", "security", "vault"
]
---

Handling manual credentials to databases can be a tedious work and it can also lead to security issues. Let's check how we can generate dynamic username and passwords for Postgres using the secret manager Vault on a Go REST API.
<!--more-->

When we write boilerplate code to handle database connections we always make the same questions where to store the username/password. There are two most common options: having a configuration file or passing them as environment variables.
