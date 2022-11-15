# Fixing the security problem in mod-configuration

<!-- md2toc -l 2 fixing-mod-configuration.md -->



## Introduction

Early in FOLIO history, [`mod-configuration`](https://github.com/folio-org/mod-configuration) was created as a repository for all configuration information. It provides a simple API where a compound key consisting of `module`, `configName`, `code` (a sort of-second-order facet for the configuration name) and optionally `user` is mapped to a string value (sometimes a JSON string). Each entry may optionally have a textual description and a boolean indiction of whether the entry is enabled. It is possible to search for relevant entries using CQL.

`mod-configuration` is general and flexible, and has mostly worked well in practice. However, it has an important security hole: as explained at [UXPROD-3018 (Distributed Configuration)](https://issues.folio.org/browse/UXPROD-3018), modules cannot specify which permissions should govern access to which configuration entries, so that any user with access to one entry has access to them all. A user who has been granted permission to know whether or not tags are enabled thereby also has permission to see the LDP database connection details. Indeed, the contents of _all_ `mod-configuration` entries can be viewed using the developer settings at [`/settings/developer/okapi-configuration`](https://folio-snapshot.dev.folio.org/settings/developer/okapi-configuration).

Because of this issue, [as of 28 March 2022](https://github.com/folio-org/mod-configuration/commit/812c7d15fcb264359c89c2d5b43696f7c27b9462), `mod-configuration` is [deprecated](https://github.com/folio-org/mod-configuration/blob/master/README.md#deprecation):

> This module is deprecated. Please do not add new configuration values to this module.
>
> Consider using standard CRUD APIs to store configuration and settings values in the storage module they belong to. This allows to cache the value and invalidate the cache if the value gets changed.

Can `mod-configuration` be made secure? Perhaps there is a way to rehabilitate it.



## Proposal

At present, one of the fields in each configuration entry indicates the module that the entry belongs to. For example, the `mod-login-saml` stores the identity provider URL in an entry that has `module`=`LOGIN-SAML`, `configName`=`saml`, `code`=`idp.url`, and with a value of (for example) `https://samltest.id/saml/idp`.

The configuration code could be expanded to examine permissions whose names are derived from the specified module -- in this case, for example,
* `configuration.byModule.LOGIN-SAML.read` to read values
* `configuration.byModule.LOGIN-SAML.write` to write values

This would probably address the great majority of security concerns, but there are some wrinkles that would need to be addressed.


### Permission-name mangling

There is no well-established convention for how module identities are specified in the `module` field of a configuation entry. Current entries include `@folio/users`, `BULKEDIT`, `GOBI`, and `LOGIN-SAML`. Evidently, some module names are scoped to organization namespaces, some are not; some are lower-case and some are capitalized; some have underscored between words, and others omit these.

Some of these characters will likely not be usable in permission names: for example, `configuration.byModule.@folio/users.read` may not work. If this is so, then we will need a convention for converting the module-name tags used in `mod-configuration` into a form that can be used in permission names: for example, downcasing all capital letters and transforming everything but  alphanumerics and hyphens into underscores, which would give us permission names like `configuration.byModule.login-saml.read` and `configuration.byModule._folio_users.read`.


### Desired permissions

Okapi will only pass `mod-configuration` the desired permissions that it actively specifies that it wants, in the `permissionsDesired` element of the relevant handler definition in its module descriptor. Obviously the maintainer of that module descriptor cannot know in advance which modules will use it, so it cannot list all the relevant `configuration.byModule.MODULE.read` and `configuration.byModule.MODULE.write` permissions.

But this turns out not to be a problem. Whatever desired-permission string is specified in a module-descriptor handler declaration, Okapi passes it blindly through to `mod-authorization` -- and that module already does wildcard expansion. As a result, we can use wilcards in desired-permission names.

So the `mod-configuration` module descriptor can specify:

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

(Optionally, we could also take this opportunity to remove `code`, the unnecessary third facet to configuration-entry names, so that each entry is identified only by the combination of `module` and `configName`.)



