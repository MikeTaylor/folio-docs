# Configuration and customization in FOLIO

<!-- md2toc -l 2 customization.md -->
* [Introduction](#introduction)
* [The current situation](#the-current-situation)
    * [mod-configuration](#mod-configuration)
    * [Work towards other approaches](#work-towards-other-approaches)
    * [Module-specific configuration stores](#module-specific-configuration-stores)
* [Requirements for a new configuration solution](#requirements-for-a-new-configuration-solution)
* [Concrete proposals](#concrete-proposals)
    * [Independent implementations](#independent-implementations)
    * [A Java library](#a-java-library)
    * [A FOLIO module](#a-folio-module)
* [Tentative conclusion](#tentative-conclusion)



## Introduction

The FOLIO system has numerous uses for persistent configuration, but no very good solution for how to store this information. As a result, efforts towards making FOLIO customizable have stalled on the problem of there being no suitable infrastructure to support the desired customizations.

Among the configuration information currently stored in a FOLIO is:
* Which FOLIO users to protect from editing
* The statuses available for the bulk-edit facility
* Various aspects of how the OAI-PMH server should act
* Details about the SMTP server that FOLIO uses to send emails
* Whether tags are enabled
* The set of GitHub repositories holding app metadata for the App Manager
* Record limits and table availability in the LDP app
* Configuration of the back-end LDP database in the LDP app
* Location of the GitHub repository holding saved queries for the LDP app

Most of the examples in this list are stored in `mod-configuration` (see below), but the last two are stored in LDP's own key-value configuration store. The fact that LDP's stored configuration is split across multiple places is indicative of a lack of Grand Unified Theory.



## The current situation


### mod-configuration

Early in FOLIO history, [`mod-configuration`](https://github.com/folio-org/mod-configuration) was created as a repository for all configuration information. It has been widely used, including for most of the purposes listed above. It provides a simple API where a compound key consisting of `module`, `configName`, `code` (a sort of-second-order facet for the configuration name) and optionally `user` is mapped to a string value (sometimes a JSON string). Each entry may optionally have a textual description and a boolean indiction of whether the entry is enabled. It is possible to search for relevant entries using CQL.

`mod-configuration` is general and flexible, and has mostly worked well in practice. However, it has an important security hole: as explained at [UXPROD-3018 (Distributed Configuration)](https://issues.folio.org/browse/UXPROD-3018), modules cannot specify which permissions should govern access to which configuration entries, so that any user with access to one entry has access to them all. A user who has been granted permission to know whether or not tags are enabled thereby also has permission to see the LDP database connection details. Indeed, the contents of _all_ `mod-configuration` entries can be viewed using the developer settings at [`/settings/developer/okapi-configuration`](https://folio-snapshot.dev.folio.org/settings/developer/okapi-configuration).

Because of this issue, [as of 28 March 2022](https://github.com/folio-org/mod-configuration/commit/812c7d15fcb264359c89c2d5b43696f7c27b9462), `mod-configuration` is [deprecated](https://github.com/folio-org/mod-configuration/blob/master/README.md#deprecation). Unhelpfully, it is not deprecated _in favour of_ anything. Developers are merely told not to use it, and to roll their own configuration mechanisms. (This is known as "distributed configuration").


### Work towards other approaches

After the FOLIO security team identified the permissions hole in `mod-configuration` their audit of 2020, Craig McNally filed [UXPROD-3018](https://issues.folio.org/browse/UXPROD-3018) summarising the problem and briefly laying out some possible approaches to addressing it. The Technical Council agreed in their meeting of 2 September 2020 that [the issue should be addressed](https://wiki.folio.org/display/TC/2020-09-02+Meeting+notes) but at the time of writing there is no practicable proposal.

Craig McNally wrote some design documents on the FOLIO Wiki on what might supersede mod-`configuration`:
* [Distributed Configuration](https://wiki.folio.org/display/DD/Distributed+Configuration)
* [Distributed Configuration via Multiple Interfaces and Scope](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Multiple+Interfaces+and+Scope)
* [Distributed Configuration via Namespace](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Namespace)
* [Distributed Configuration via Path](https://wiki.folio.org/display/DD/Distributed+Configuration+via+Path)

The issue to replace `mod-configuration` with distributed configuration was filed over a year and a half ago, but seems to have become mired in discussions that spiralled out into all kinds of meta-configurability, and other security-related issues. Possibly as a result, no concrete progress seems to have been made with it since then.


### Module-specific configuration stores

In the absence of a proper replacement for `mod-configuration`, configuration has become a bit of a Wild West, with different modules taking different approaches.

Some, such as [`mod-app-manager`](https://github.com/MikeTaylor/mod-app-manager) are going ahead and using `mod-configuration` despite its flaws. (This is justifiable only when it can be established that important secrets cannot leak in this way. In the case of `mod-app-manager` the only secrets are GitHub tokens giving read-only WSAPI access to repositories that are already publicly visible using the Web UI.)

Others, such as [`mod-ldp`](https://github.com/folio-org/mod-ldp/), are providing their own configuration storage protected by their own permissions. While this solves the permissions problem, it does tend to result in numerous similar but different re-implementations of different subsets of `mod-configuration`'s functionality. For example, FOLIO now has (among other configuration facilities):

* `/inventory/config` (for mod-inventory)
* `/converter-storage/forms/configs` (for mod-data-import-converter-storage)
* `/batch-voucher-storage/export-configurations` (for mod-invoice-storage)
* `/orders-storage/configuration` (for mod-orders-storage)
* `/eventConfig` (for mod-event-config)

Note the use of both `config` and `configs`; and both `configuration` and `configuration`; as well as `eventConfig`. If even the pathnames are this disparate, it's anyone's guess how much difference there is between the details of the APIs they provide.

In at least some cases, the facilities that have been built are not really sufficiently powerful to meet the needs of the relevant applications. For example, `mod-ldp` has implemented a very simple configuration facility at `/ldp/config` that is only a key-value store for srtrings. It is as a result of this that we have [the compromise solutions proposed for `ui-ldp`'s configuration needs](https://github.com/folio-org/ui-ldp/blob/master/doc/templated-sql-queries.md#handling-configuration-storage) in the area of templated SQL queries.

It seems clear that the current situation is not really acceptable. There is a strong practical need a new unified approach to configuration.



## Requirements for a new configuration solution

If we are to somehow provide a unified approach to configuration within FOLIO, then it will need to be sufficiently general to meet the configuration and customization requirements of all modules that use or want to use these  facilities. As a lower bound, we can consider the requirements of the LDP app. This needs to use configuration for several rather different purposes. It must store:

1. Connection details (URL, username, password) for the underlying LDP database.

2. A list of tables that should be disabled for querying. This can be encoded in a single configuration entry, as a string that is a JSON array.

3. A set of numeric parameters, such as the default number of records to show and the maximum number to export.

4. Connection details for a small set of GitHub repositories that will contain SQL query templates: repository owner, repository name, branch and access token for each.

5. A set of potentially many saved "JSON queries" that can be retrieved and run. ("JSON query" here means a query expressed in a structured form that can be maintained using the LDP Builder app, as opposed to an SQL string.)

For #1 and #2, we need to store system-wide defaults, and do not want users to be able to override these.

For #1 and #3, we could store all the related parameters together as values in a single JSON object, or we might elect to store each in its own configuration entry.

For #3, we must store system-wide values but we might also wish to allow those values to be overridden for individual users.

In #4, a new requirement appears: the need to store multiple records and maintain them as a list, rather than as a single record. Not only this, but while a system-wide set of entries is required, it would not be unreasonable for an individual user to want to use a modified list. What the semantics should be for overriding such a list is itself an open question, even before we get to the matter of how to implement the desired semantics.

Finally, #5 is similar to #4 except that the number of stored entries may be much greater -- potentially enough that we would not want to present a list of all of them on a single page, so that some form of searching and paging would be required. Also, the semantics of per-user overrides will be different from  case 4. Probably we would need to maintain a system-wide set of queries and allow users to add further private queries of their own.

The full set of requirements, then, is both sophisticated and subtle. While most of the requirements here can be addressed by suitable technical infrastructure, the question of what it means for an individual user to override a site-wide list of configuration entries seems to be different for different kinds of resources: in some cases we might want to allow a user to override the whole of the system-supplied default, i.e. as soon as a user creates an entry of his own, the system default entries become invisible to him. In others we might want to support per-user editing of the list: the addition of entries visible only to the user, the overriding of system-wide entries with modified versions visible only to the user, and even the removal of entries from the user's view of the system-wide list.



## Concrete proposals

There are three possible ways to provide this configuration/customization functionality to all the modules that need it.


### Independent implementations

One possibility is just to keep doing what we've been doing: each back-end module independently implements its own configuration facilities, with all the inconsistency and duplication of effort that this entails.

This option is mentioned only for completeness.

Given that this seems like the worst option, why is it the one we as a community are currently pursuing? This seems to be a group-action problem. For any given module, it's quicker to hack in the configuration requirements needed _right now_ than to wait for, or contribute towards, a more general solution. But once the problem is addressed properly, the whole community will benefit.


### A Java library

One way to address a shared approach to configuration is to provide a Java library that implements the facilities outlined above, persisting configuration entries to the FOLIO PostSQL database. This could be included by any Java-based module that needs configuration, and access to the configuration facilities could be provided as part of the module's own API, under its own WSAPI path, protected by its own permissions.

The key advantage would be that a single codebase, shared by all modules, would be the focus of work in developing additional features, security testing, documentation, etc.

One disadvantage is that we presently have Java-based modules built on three different foundations:
[RAML Module Builder (RMB)](https://github.com/folio-org/raml-module-builder),
[Spring Way](https://github.com/folio-org/folio-spring-base),
and [`folio-vertx-lib`](https://github.com/folio-org/folio-vertx-lib). Integration of a configuration library would likely be rather different in each of these environments, meaning that we would possibly end up with three small "bridging" libraries -- one per foundation library -- as well as the main library.

A more serious disadvantage is that such a library would be available only for modules written in Java. Although Java dominates the present module ecosystem, FOLIO is explicitly a polyglot system, and configuration support for modules written in other languages is desirable. In particular, some UI modules use configuration services directly, and would not able to do so unless they were furnished by means of a WSAPI.


### A FOLIO module

A more general way to provide configuration services is by means of a module, in keeping with FOLIO's module architecture.

The key advantage to this approach is that such a configuration facility is available to all modules, both in the UI and on the back end.

Another signficant advantage is that the core of such a module already exists, in our old friend [`mod-configuration`](https://github.com/folio-org/mod-configuration). Fixing a problem in an existing module is an order of magnitude less work than creating a new model, and it seems that it's possible to [make `mod-configuration` secure](fixing-mod-configuration.md) with relatively small changes.



## Tentative conclusion

Given the current situation, where `mod-configuration` exists and continues to be widely used despite its state of deprecation, simply fixing its security hole seems like the easiest _and_ most effective solution. This module's [existing search facilities](https://github.com/folio-org/mod-configuration#query-syntax) -- which admittedly are in need of much better documentation -- should suffice to support most or all of the scenarios outlined above: for example, it's possible to search configuration entries in a way that limits results to those entries associated with a particular user.

Read on to the details of [Fixing the security problem in mod-configuration](fixing-mod-configuration.md).



