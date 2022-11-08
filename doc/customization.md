# Configuration and customization in FOLIO

<!-- md2toc -l 2 customization.md -->
* [Introduction](#introduction)
* [The current situation](#the-current-situation)
    * [mod-configuration](#mod-configuration)
    * [Module-specific configuration stores](#module-specific-configuration-stores)
* [Requirements for a new configuration solution](#requirements-for-a-new-configuration-solution)
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

Early in FOLIO history, [`mod-configuration`](https://github.com/folio-org/mod-configuration) was created as a repository for all configuration information. It has been widely used, including for most of the purposes listed above. It provides a simple API where a compound key consisting of `module`, `configName`, `code` (a sort of-second-order facet for the configuration name) and optionally `user` is mapped to a string value (sometimes a JSON string).

However, `mod-configuration` has an important security hole: as explained at [UXPROD-3018 (Distributed Configuration)](https://issues.folio.org/browse/UXPROD-3018), modules cannot specify which permissions should govern access to which configuration entries, so that any user with access to one entry has access to them all. Indeed, the contents of `mod-configuration` can be viewed using the developer settings at [`/settings/developer/okapi-configuration`](https://folio-snapshot.dev.folio.org/settings/developer/okapi-configuration).

Because of this issue, [as of 28 March 2022](https://github.com/folio-org/mod-configuration/commit/812c7d15fcb264359c89c2d5b43696f7c27b9462), `mod-configuration` is [deprecated](https://github.com/folio-org/mod-configuration/blob/master/README.md#deprecation):

> This module is deprecated. Please do not add new configuration values to this module.
>
> Consider using standard CRUD APIs to store configuration and settings values in the storage module they belong to. This allows to cache the value and invalidate the cache if the value gets changed.

Unhelpfully, it is not deprecated _in favour of_ anything. Developers are merely told not to use it, and to roll their own configuration mechanisms. (This is known as "distributed configuration").

The FOLIO Wiki has  some design documents on what might supersede mod-`configuration`:
* [Distributed Configuration](https://wiki.folio.org/display/DD/Distributed+Configuration)
* [Distributed Configuration via Multiple Interfaces and Scope](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Multiple+Interfaces+and+Scope)
* [Distributed Configuration via Namespace](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Namespace)
* [Distributed Configuration via Path](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Path)

But the issue to replace `mod-configuration` with distributed configuration was filed over a year and a half ago, and no real progress seems to have been made with it since then.

It may be possible to [make `mod-configuration` secure](fixing-mod-configuration.md). But if that approach is not taken, then we will need to do something different.


### Module-specific configuration stores

In the absence of a proper replacement for `mod-configuration`, configuration has become a bit of a Wild West, with different modules taking different approaches.

Some, such as [`mod-app-manager`](https://github.com/MikeTaylor/mod-app-manager) are going ahead and using `mod-configuration` despite its flaws. (This is best done only when it can be established that important secrets cannot leak in this way.)

Others, such as [`mod-ldp`](https://github.com/folio-org/mod-ldp/), are providing their own configuration storage protected by their own permissions. While this solves the permissions problem, it does tend to result in numerous similar bit different re-implementations of different subsets of `mod-configuration`'s functionality. In at least some cases, the facilities that have been built are not really sufficiently powerful to meet the needs of the relevant applications -- hence for example the [compromise solutions proposed for `ui-ldp`'s configuration needs](https://github.com/folio-org/ui-ldp/blob/master/doc/templated-sql-queries.md#handling-configuration-storage).

It seems clear that the current situation is not really acceptable. Unless a way can be found to make `mod-configuration` sufficiently secure, we will need a new unified approach to configuration.



## Requirements for a new configuration solution

XXX



## A proposal

XXX



