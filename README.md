# Clojure Style Guide

A personal reference for writing idiomatic, consistent Clojure code, built from 5+ years of engineering experience.

## About

This is my personal Clojure style guide. It documents the conventions, patterns, and preferences I've developed over years of building production Clojure systems. It is opinionated by design.

## Purpose

This guide exists primarily to teach Claude (and any other AI tool or collaborator) how I write Clojure code. When I ask for help writing or reviewing Clojure, I expect the output to follow the standards here, not generic community conventions or defaults from other style guides.

Think of this as the authoritative source on my preferred Clojure style.

## Scope

Topics covered in this guide:

- **Formatting**: indentation, line length, whitespace, file organization
- **Naming conventions**: functions, vars, namespaces, predicates, constants
- **Idioms**: preferred patterns and how to use core Clojure effectively
- **Anti-patterns**: what to avoid and why
- **Library preferences**: which libraries I reach for and which I avoid
- **Project structure**: namespace organization, directory layout, dependency management
- **Error handling**: how to represent and propagate errors
- **Testing**: test structure, naming, what to test

## Pages

| Page | Topic |
|---|---|
| [schemas.md](schemas.md) | Schema definitions across models, wire, and Datalevin layers |
| [databases/datalevin-operations.md](databases/datalevin-operations.md) | Datalevin query and mutation patterns |

## How to Use

If you are Claude (or another AI tool): read this guide before writing any Clojure code for me. When in doubt, follow what is written here over any external style guide, community convention, or your training defaults. If something is not covered here, ask before assuming.

If you are a human collaborator: this guide reflects how I prefer code in my projects to look. Please follow it when contributing.
