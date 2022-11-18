# Fixing the security problem in mod-configuration

<!-- md2toc -l 2 fixing-mod-configuration.md -->
* [Introduction](#introduction)
* [Proposal](#proposal)
    * [Permission-name restrictions](#permission-name-restrictions)
    * [Desired permissions](#desired-permissions)
    * [Backward-compatibility and migration](#backward-compatibility-and-migration)



## Introduction

Early in FOLIO history, [`mod-configuration`](https://github.com/folio-org/mod-configuration) was created as a repository for all configuration information. It provides a simple API where a compound key consisting of `module`, `configName`, `code` (a sort of-second-order facet for the configuration name) and optionally `user` is mapped to a string value (sometimes a JSON string). Each entry may optionally have a textual description and a boolean indiction of whether the entry is enabled. It is possible to search for relevant entries using CQL.

`mod-configuration` is general and flexible, and has mostly worked well in practice. However, it has an important security hole: as explained at [UXPROD-3018 (Distributed Configuration)](https://issues.folio.org/browse/UXPROD-3018), modules cannot specify which permissions should govern access to which configuration entries, so that any user with access to one entry has access to them all. A user who has been granted permission to know whether or not tags are enabled thereby also has permission to see the LDP database connection details. Indeed, the contents of _all_ `mod-configuration` entries can be viewed using the developer settings at [`/settings/developer/okapi-configuration`](https://folio-snapshot.dev.folio.org/settings/developer/okapi-configuration).

Because of this issue, [as of 28 March 2022](https://github.com/folio-org/mod-configuration/commit/812c7d15fcb264359c89c2d5b43696f7c27b9462), `mod-configuration` is [deprecated](https://github.com/folio-org/mod-configuration/blob/master/README.md#deprecation):

> This module is deprecated. Please do not add new configuration values to this module.
>
> Consider using standard CRUD APIs to store configuration and settings values in the storage module they belong to. This allows to cache the value and invalidate the cache if the value gets changed.

Can `mod-configuration` be made secure? Perhaps there is a way to rehabilitate it.



## Proposal

At present, one of the fields in each configuration entry indicates the module that the entry belongs to. For example, `mod-login-saml` stores the identity provider URL in an entry that has `module`=`LOGIN-SAML`, `configName`=`saml`, `code`=`idp.url`, and with a value of (for example) `https://samltest.id/saml/idp`. In practice, the association of a configuration entry with a module has no consequences: the `module` field is only a facet of the key, along with `configName` and `code`.

We propose to replace this `module` field with `scope`. This would be a short machine-readable string, composed only of letters, digits, underscores and hyphens, indicating the permission scope under which the entry is viewed and managed. Scopes may be module-wide (e.g. `login-saml` could be a scope) or may be more specific to particular areas of functionality within a module (e.g. `oaipmh-general` and `oaipmh-admin`.

The configuration code would then be enhanced to examine permissions whose names are derived from the entry's scope -- in this case, for example,
* `configuration.byScope.login-saml.read` to read values
* `configuration.byScope.login-saml.write` to write values

It would be the responsibility of a module that uses a scope to define the permissions that are in it: for example, the permissions above would be defined in the module descriptor of `mod-login-saml`.

This would address the great majority of security concerns: each module would manage its own scopes, each with its own pair of permissions for read-only and read/write access to the coniguration store. A user could be assigned any combination of permissions. A single user might have permission to read the configuration of which users are suppressed from editing, to write the bulk-edit expiration period, and not to access the OAI-PMH server settings at all.

There are however some wrinkles that would need to be addressed.


### Permission-name restrictions

In the present version of `mod-configuration` there is no well-established convention for how module identities are specified in the `module` field of a configuation entry. Current entries include `@folio/users`, `BULKEDIT`, `GOBI`, and `LOGIN-SAML`. Evidently, some module names are scoped to organization namespaces, some are not; some are lower-case and some are capitalized; some have underscored between words, and others omit these.

Some of the characters currently used in `module` fields are likely not to be usable in permission names: for example, a permission named `configuration.byScope.@folio/users.read` may not work. It is for this reason that the new `scope` field is limited to letters, digits, underscores and hyphens: the characters that are typically used in the facets of permission names.


### Desired permissions

Okapi will only pass `mod-configuration` the desired permissions that it actively specifies that it wants, in the `permissionsDesired` element of the relevant handler definition in its module descriptor. Obviously the maintainer of that module descriptor cannot know in advance which modules will use it, so it cannot list all the relevant `configuration.byScope.SCOPE.read` and `configuration.byScope.SCOPE.write` permissions.

But this turns out not to be a problem. Whatever desired-permission string is specified in a module-descriptor handler declaration, Okapi passes it blindly through to `mod-authorization` -- and that module already does wildcard expansion. As a result, we can use wildcards in desired-permission names.

So the `mod-configuration` module descriptor can specify:

	"handlers": [
	  {
	    "methods": ["GET"],
	    "pathPattern": "/configurations/byScope/{id}",
	    "permissionsRequired": [
	      "configuration.entries.item.get"
	    ],
	    "permissionsDesired": [
	      "configuration.byScope.*.read"
	    ],
	  },
 
To have all permissions matching the pattern `configuration.byScope.*.read` forwarded to it.


### Backward-compatibility and migration

The replacement of the `module` field with `scope` is a breaking change. Even if we re-used the existing `module` field with modified semantics, the requirement for a user to have a new module-specific (i.e. scope-specific) permission in order to use `mod-configuration` would constitute a breaking change: not just a theoretical one, but a change that would in practice break a lot of things.

For this reason, we would keep the old API as it is, and put the new one on a different WSAPI path -- for example, the 
`/configurations/byScope` suggested in the module-descriptor fragment above, or perhaps something more terse such as `/config`.

(Optionally, we could also take this opportunity to remove `code`, the unnecessary third facet to configuration-entry names, so that each entry is identified only by the combination of `scope` and `configName`.)

The old, insecure configuration API would remain in play for the time being, and the addition of the new API would be a non-breaking change meriting only a minor version bump in `mod-configuration`.

Over time, we would expect client modules to gradually move away from the old API to the new -- a relatively simple process which we would document in a short HOWTO document. Those clients that are are happy for their information to be globally available as in the present system can use a `global` scope defined by `mod-configuration` itself.

At some point, we would announce the sunsetting of the old configuration WSAPI, and remaining clients would have a limited time to move away from it onto the new WSAPI. At the end of this period, the old WSAPI would be removed, and a new major version of `mod-configuration` would be released since we would now -- only at this point -- have a backwards-incompatible change.


