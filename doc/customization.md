# Configuration and customization in FOLIO

<!-- md2toc -l 2 customization.md -->
* [Introduction](#introduction)
* [The current situation](#the-current-situation)
    * [mod-configuration](#mod-configuration)
    * [Module-specific configuration stores](#module-specific-configuration-stores)
* [Digression: can `mod-configuration` be made secure?](#digression-can-mod-configuration-be-made-secure)
    * [Permission-name mangling](#permission-name-mangling)
    * [Desired permissions](#desired-permissions)
    * [Backward-compatibility](#backward-compatibility)
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


### Module-specific configuration stores

In the absence of a proper replacement for `mod-configuration`, configuration has become a bit of a Wild West, with different modules taking different approaches.

Some, such as [`mod-app-manager`](https://github.com/MikeTaylor/mod-app-manager) are going ahead and using `mod-configuration` despite its flaws. (This is best done only when it can be established that important secrets cannot leak in this way.)

Others, such as [`mod-ldp`](https://github.com/folio-org/mod-ldp/), are providing their own configuration storage protected by their own permissions. While this solves the permissions problem, it does tend to result in numerous similar bit different re-implementations of different subsets of `mod-configuration`'s functionality. In at least some cases, the facilities that have been built are not really sufficiently powerful to meet the needs of the relevant applications -- hence for example the [compromise solutions proposed for `ui-ldp`'s configuration needs](https://github.com/folio-org/ui-ldp/blob/master/doc/templated-sql-queries.md#handling-configuration-storage).

It seems clear that the current situation is not really acceptable. Unless a way can be found to make `mod-configuration` sufficiently secure, we will need a new unified approach to configuration.



## Digression: can `mod-configuration` be made secure?

Perhaps there is a way to rehabilitate `mod-configuration`.

At present, one of the fields in each configuration entry indicates the module that the entry belongs to. For example, the `mod-login-saml` stores the identity provider URL in an entry that has `module`=`LOGIN-SAML`, `configName`=`saml`, `code`=`idp.url`, and with a value of (for example) `https://samltest.id/saml/idp`.

The configuration code could be expanded to examine permissions whose names are derived from the specified module -- in this case, for example,
* `configuration.byModule.LOGIN-SAML.read` to read values
* `configuration.byModule.LOGIN-SAML.write` to write values

This would probably address the great majority of security concerns, but there are some wrinkles that would need to be addressed.


### Permission-name mangling

There is no well-established convention for how module identities are specified in the `module` field of a configuation entry. Current entries include `@folio/users`, `BULKEDIT`, `GOBI`, and `LOGIN-SAML`. Evidently, some module names are scoped to organization namespaces, some are not; some are lower-case and some are capitalized; some have underscored between words, and others omit these.

Some of these characters will likely not be usable in permission names: for example, `configuration.byModule.@folio/users.read` may not work. If this is so, then we will need a convention for converting the module-name tags used in `mod-configuration` into a form that can be used in permission names: for example, downcasing all capital letters and transforming everything but  alphanumerics and hyphens into underscores, which would give us permission names like `configuration.byModule.login-saml.read` and `configuration.byModule._folio_users.read`.


### Desired permissions

Okapi will only pass `mod-configuration` the desired permissions that it actively specifies that it wants, in the `permissionsDesired` element of a the relevant handler definition in its module descriptor. Obviously the maintainer of that module descriptor cannot know in advance which modules will use it, so it cannot list all the relevant `configuration.byModule.MODULE.read` and `configuration.byModule.MODULE.write` permissions. In order to meaningfully support the proposed approach to securing configuration, Okapi (or perhaps `mod-authorization` or something related) would need to be modifield to support wildcards in desired permissions. Then the `mod-configuration` module descriptor could specify:

	"handlers": [
	  {
	    "methods": ["GET"],
	    "pathPattern": "/configurations/byModule/{id}",
	    "permissionsRequired": [
	      "configuration.entries.item.get"
	    ],
	    "permissionsDesired": [
	      "configuration.byModule.*.read"
	    ],
	  },
 
To have all permissions matching the pattern `configuration.byModule.*.read` forwarded to it.


### Backward-compatibility

The requirement for a user to have a new module-specific permission in order to use `mod-configuration` would constitute a breaking change -- and not just a theoretical one, but a change that would in practice break a lot of things.

For this reason, we would keep the old API as it is, and gradually move away from it to the new API, which would be on a different path -- for example, the 
`/configurations/byModule` suggested in the module-descriptor fragment above.

Since client modules would need to opt into the new API (by defining their desired permissions for read and write, and by switching to the new API path), it would make sense also to change the module-name convention used in the configuration entries at the new endpoint. We could canonicalize on a convention where, for example, we always use `@NAMESPACE/NAME` for UI modules and `mod-NAME` for backend modules, yielding `@folio/users` and `mod-users`.

(Optionally, we could also take this opportunity to remove `code`, the unnecessary third facet to configurate-entry names, so that each entry is identified only by the combination of `module` and `configName`.)


## Requirements for a new configuration solution

XXX



## A proposal

XXX



