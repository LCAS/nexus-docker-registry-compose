# Docker Compose Configuration

This repository contains a Docker Compose configuration to set up various services including Nexus repositories and Zrok.

## Services

### Nexus

A repository manager for storing and managing artifacts, mainly used for docker registry for lcas.lincoln.ac.uk

### Nexus AOC

An additional Nexus repository manager instance for AOC-specific artifacts (docker images), deployed at aoccr.zrok.lcas.group

### Zrok Repository

A service for sharing reserved repositories using Zrok.

### Zrok Registry

A service for sharing reserved container registries using Zrok.

### Registry Frontend LCAS

A web-based frontend for browsing and managing Docker images in the registry.
