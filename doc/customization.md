# Configuration and customization in FOLIO

<!-- md2toc -l 2 customization.md -->
* [Introduction](#introduction)
* [The current situation](#the-current-situation)
    * [`mod-configuration`](#mod-configuration)
    * [Module-specific configuration stores](#module-specific-configuration-stores)
* [Requirements](#requirements)
* [A proposal](#a-proposal)



## Introduction

The FOLIO system has numerous uses for persistent configuration, but no very good solution for how to store this information. As a result, efforts towards making FOLIO customizable have stalled on the problem of their being no infrasructure to support the desired customizations.

Among the configuration information currently stored in a FOLIO is:
* Which FOLIO users to prohibit the editing of
* The statuses available for the bulk-edit facility
* Various aspects of how the OAI-PMH server should act
* Details about the SMTP server that FOLIO uses to send emails
* Whether tags are enabled
* The set of GitHub repositories holding app metadata for the App Manager
* Record limits and table availability in the LDP app
* Configuration of back-end LDP database in the LDP app
* Location of GitHub repository holding saved queries for the LDP app

Most of the examples in this list are stored in `mod-configuration` (see below), but the last two are stored in LDP's own key-value configuration store. The fact that LDP's stored configuration is split across multiple places is indicative of a lack of Grand Unified Theory.



## The current situation


### mod-configuration

Early in FOLIO history, [`mod-configuration`](https://github.com/folio-org/mod-configuration) was created as a repository for all configuration information. it has been widely used, including for most of the purposes listed above. It provides a simple API where a compound key consisting of module, configName and optionally user is mapped to a string value (sometimes a JSON string) and optionally a description.

However, [as of 28 March 2022](https://github.com/folio-org/mod-configuration/commit/812c7d15fcb264359c89c2d5b43696f7c27b9462), `mod-configuration` is [deprecated](https://github.com/folio-org/mod-configuration/blob/master/README.md#deprecation):

> This module is deprecated. Please do not add new configuration values to this module.
>
> Consider using standard CRUD APIs to store configuration and settings values in the storage module they belong to. This allows to cache the value and invalidate the cache if the value gets changed.

Unhelpfully, it is not deprecated _in favour of_ anything. Developers are merely told not to use it, and to roll their own configuration mechanisms.

XXX See also:
* https://issues.folio.org/browse/UXPROD-3018
* https://wiki.folio.org/display/DD/Distributed+Configuration
* https://wiki.folio.org/display/DD/Distributed+Configuration+via+Multiple+Interfaces+and+Scope
* https://wiki.folio.org/display/DD/Distributed+Configuration+via+Namespace
* https://wiki.folio.org/display/DD/Distributed+Configuration+via+Path

XXX The contents of `mod-configuration` can be viewed using the developer settings at [`/settings/developer/okapi-configuration`](https://folio-snapshot.dev.folio.org/settings/developer/okapi-configuration).


### Module-specific configuration stores

XXX



## Requirements

XXX



## A proposal

XXX



