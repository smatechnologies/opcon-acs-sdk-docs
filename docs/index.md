---
slug: '/'
sidebar_label: 'ACS (Agentless Connector System) SDK'
---

# ACS SDK Documentation

The **ACS (Agentless Connector System) SDK** allows developers, both within SMA Technologies and without, to quickly build software plugins which allow SMA Technologies products to integrate with external applications.

# General Concepts & Terminology

## External Application
An **External Application** is any software, tool, or service which exists outside of a core SMA Technologies product. It can be as complex as a complete operating system or as simple as a command-line utility script - the only requirement is that it offers some mechanism to perform functionality programmatically.

## Integration
An **Integration** is a piece of software which is written to translate instructions from SMA Technologies products in order to perform tasks within an External Application and then report back with the results of those tasks. The ACS SDK offers a flexible and powerful Framework to facilitate construction of these Integrations. It encapsulates all business logic which is necessary to execute it's associated tasks in addition to describing it's configuration options.
