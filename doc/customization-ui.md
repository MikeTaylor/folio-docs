# A user-interface for FOLIO customization


<!-- md2toc -l 2 customization-ui.md -->
* [Introduction](#introduction)
* [User interface](#user-interface)
* [Configuration](#configuration)
* [Interaction with other apps](#interaction-with-other-apps)


## Introduction

There are many reasons why we need [a secure general-purpose configuration solution](customization.md) for FOLIO. Once we have created such a solution -- for example, by [fixing the security problem in `mod-configuration`](fixing-mod-configuration.md) -- we will be in a position to create a powerful and general Configuration app.

In this document, we will briefly envision what such a thing might look like, what it might need in the way of configuration, and how it might interact with other modules.


## User interface

We can imagine a new FOLIO app, Customise -- implemented by means of a UI module, `ui-customization` -- which acts as an app within FOLIO just like Users, Inventory and Agreements. It would appear in the FOLIO top bar, or perhaps as an area under the Settings app.

Like all FOLIO apps, access to it would be granted by giving a user a relevant permission. At a base level, this would enable the user to read those configuration entries that he has permissions for, and to create per-user overrides for those entries.

A second permission would give an administrative user the ability to view and edit system-wide configuration entries (again subject to relevant permissions for the specific entries).

In general, a per-user setting would specify that user's value of the relevant configuration entry, overriding the system-wide value with the corresponding module, name and code.

**NOTE.**
We will need to give more thought to how mod-configuration handles global and per-user entries, limiting some users to viewing and/or editing only their own personal entries -- not those of other users, or global entries. We will want mod-configuration to enforce this, not just make it a convention suggested by the UI.


## Configuration

The Customize app will itself be configured by a set of records describing what configuration entries exist, what they do, which can only be set by an administrator, which can be overriden on a per-user basis, etc. (We will discuss the source of these records below.)

Each record will need fields along the following lines:

* `displayName` -- The name of the configuration entry as displayed on screen.
* `description` -- Optionally, a longer description of that the entry represents, and how different possible values would affect the customization of eFOLIO.
* `module`, `configName` and `code` -- the values of these fields to use when reading and writing the entry to `mod-configuration`. These three fields together (or two of them, when the optional `code` is omitted) constitute the machine-readable address of the entry in the configuration database.
* `type` -- The type of the value that the field should take. Candidate values include:
  * `string` -- Any string.
  * `number` -- A numeric constant.
  * `boolean` -- A Truth value, `true` or `false`.
  * `userId` -- The UUID of a user within the system. While these could be edited as strings, doing so would be cumbersome, while specifying the `userId` type allows the customization UI to offer a user-selection plugin as a way of editing the field.
  * `list`
* `listElementType` -- When `type` is list, this specifies the type of the elements within the list. Its value can be any of those used for `type`, except for `list`.
* `overridable` -- A boolean indication of whether the system-wide record for this customization can be overridden by individual users.

A typical record might look like this:

```
{
  "displayName": "Edit-suppressed users",
  "description": "A list of users whose records cannot be edited in the Users app.",
  "module": "@folio/users",
  "configName": "suppressEdit",
  "type": "list",
  "listElementType": "userId",
  "overridable": false
}
```

**NOTE.**
We may wish to give some thought to how to internationalize the two human-fieldable fields in this structure, `displayName` and `description`. One simple approach would be to inspect the first character of the values. If it is an asterisk (`*`), then the remainder of the string would be interpreted as the name of a translation tag whose value should be used; otherwise the whole string would be used as-is. So reasonable values of `displayName` would include "Edit-suppressed users" and "*ui-users.config.editSuppressed.displayName".


## Interaction with other apps

In early implementations of the Customize app, it will suffice to hardwire a registry of records that specifies what fields are available for customization.

But as the number of entries grows, this will quickly become cumbersome. Instead, it will be better to have each app include in its metadata a list of records describing the configuration entries that it supports. (This list could be stored as a JSON array within the `stripes` area of the UI module's package file.)

The Stripes loader would need to be modified to harvest this information from each UI module in its bundle, just as it presently does for translations, icons and other resources. (Or perhaps it would be simpler for the Customize app to pull this information out for itself.)

As well as distributing the work of maintaining customization metadata, this approach has the benefit that each participating module will maintain not only this metadata but also the associated translations.


