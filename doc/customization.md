# Configuration and customization in FOLIO

<!-- md2toc -l 2 customization.md -->
* [Introduction](#introduction)
* [The current situation](#the-current-situation)
    * [mod-configuration](#mod-configuration)
    * [Other candidate approaches](#other-candidate-approaches)
    * [Module-specific configuration stores](#module-specific-configuration-stores)
* [Requirements for a new configuration solution](#requirements-for-a-new-configuration-solution)
* [A proposal](#a-proposal)



## Introduction

The FOLIO system has numerous uses for persistent configuration, but no very good solution for how to store this information. As a result, efforts towards making FOLIO customizable have stalled on the problem of there being no infrastructure to support the desired customizations.

Among the configuration information currently stored in a FOLIO is:
* Which FOLIO users to protect from editing
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

Because of this issue, [as of 28 March 2022](https://github.com/folio-org/mod-configuration/commit/812c7d15fcb264359c89c2d5b43696f7c27b9462), `mod-configuration` is [deprecated](https://github.com/folio-org/mod-configuration/blob/master/README.md#deprecation). Unhelpfully, it is not deprecated _in favour of_ anything. Developers are merely told not to use it, and to roll their own configuration mechanisms. (This is known as "distributed configuration").

It may be possible to [make `mod-configuration` secure](fixing-mod-configuration.md). But if that approach is not taken, then we will need to do something different.


### Other candidate approaches

After the FOLIO security team identified the permissions hole in `mod-configuration` their audit of 2020, Craig McNally filed [UXPROD-3018](https://issues.folio.org/browse/UXPROD-3018) summarising the problem and briefly laying out some possible approaches to addressing it. The Technical Council agreed in their meeting of 2 September 2020 that [the issue should be addressed](https://wiki.folio.org/display/TC/2020-09-02+Meeting+notes) but at the time of writing there is no concrete proposal.

Craig McNally wrote some design documents on the FOLIO Wiki on what might supersede mod-`configuration`:
* [Distributed Configuration](https://wiki.folio.org/display/DD/Distributed+Configuration)
* [Distributed Configuration via Multiple Interfaces and Scope](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Multiple+Interfaces+and+Scope)
* [Distributed Configuration via Namespace](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Namespace)
* [Distributed Configuration via Path](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Path)

The issue to replace `mod-configuration` with distributed configuration was filed over a year and a half ago, but seems to have become mired in discussions that spiralled out into all kinds of meta-configurability, and other security-related issues. Possibly as a result, no concrete progress seems to have been made with it since then.


### Module-specific configuration stores

In the absence of a proper replacement for `mod-configuration`, configuration has become a bit of a Wild West, with different modules taking different approaches.

Some, such as [`mod-app-manager`](https://github.com/MikeTaylor/mod-app-manager) are going ahead and using `mod-configuration` despite its flaws. (This is best done only when it can be established that important secrets cannot leak in this way.)

Others, such as [`mod-ldp`](https://github.com/folio-org/mod-ldp/), are providing their own configuration storage protected by their own permissions. While this solves the permissions problem, it does tend to result in numerous similar bit different re-implementations of different subsets of `mod-configuration`'s functionality. For example, FOLIO now has (among other configuration facilities):

* `/inventory/config` (for mod-inventory/inventory-config)
* `/converter-storage/forms/configs` (for mod-data-import-converter-storage/form-configs-storage)
* `/batch-voucher-storage/export-configurations` (for mod-invoice-storage/batch-voucher-export-configuration)
* `/orders-storage/configuration` (for mod-orders-storage/configuration)
* `/eventConfig` (for mod-event-config/event_config)

Note the use of both `config` and `configs`; and both `configuration` and `configuration`; as well as `eventConfig`. If even the pathnames are this disparate, it's anyone's guess how much difference there is between the details of the APIs they provide.

In at least some cases, the facilities that have been built are not really sufficiently powerful to meet the needs of the relevant applications. For example, `mod-ldp` has implemented a very simple configuration facility at `/ldp/config` that is only a key-value store for srtrings. It is as a result of this that we have [the compromise solutions proposed for `ui-ldp`'s configuration needs](https://github.com/folio-org/ui-ldp/blob/master/doc/templated-sql-queries.md#handling-configuration-storage) in the area of templated SQL queries.

It seems clear that the current situation is not really acceptable. Unless [a way can be found to make `mod-configuration` sufficiently secure](fixing-mod-configuration.md), we will need a new unified approach to configuration.



## Requirements for a new configuration solution

If we are to somehow provide a unified approach to configuration within FOLIO, then it will need to be sufficiently general to meet the configuration and customization requirements of all modules that use or want to use these  facilities. As a lower bound, we can consider the requirements of the LDP app. This needs to use configuration for several rather different purposes:

1. xXX



## A proposal

XXX



