# Users in CYCLOPS


## The problem

* CCMS needs a notion of users so it can tie them to projects.
* But we want to use externally managed users (e.g. from FOLIO or Alma).


## Authentication

* In order to use the CYCLOPS FOLIO app, it's already necessary to authenticate at the FOLIO level.
* A CYCLOPS app in Alma would need to authenticate at the Alma level, etc.
* There is no need for CCMS to maintain its own authentication infrastructure.
* It need only keep copies of external IDs (`xid`) linking to users in the app that's using it.
* Users are tied to projects by [xid, role, projectId] triples.


## Authorization

To use FOLIO-authenticated users for CCMS operations, we have two options:
1. mod-cyclops can use FOLIO's `desired permissions` facility to determine whether the logged-in user is allowed to perform the requested operation; or
2. mod-cyclops can sent the logged-in user's `xid` (in FOLIO, a UUID) in every request, and CCMS can reject unauthorized requests.

The latter approach is preferable as it keeps all the authorization logic (users who have the role `editor` in project X are allowed to change its name) in CCMS where it belongs.

Both of these approaches require that CCMS trusts mod-cyclops: in approach 1, it must trust that mod-cyclops only makes authorized requests; in approach 2, it must trust mod-cyclops to correctly send the `xid` of the logged-in user. Of these, approach 2 is preferable because it is not vulnerable to business-logic errors in mod-cyclops.

Is it a problem for CCMS to trust mod-cyclops? I don't think so, because this is what we do all the time in FOLIO. The Postgres database trusts the back-end modules to make sensible and non-malicious changes to its contents.


