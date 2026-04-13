# Mike's module releases for FOLIO Quesnelia


## Background

General instructions can be found in
[Preparing for a new FOLIO release](https://github.com/folio-org/ui-ldp/blob/master/doc/new-folio-release.md)


## Modules

I am responsible for six modules which may need releasing. The previous release was Poppy (R2-2023), and the most recent platform-complete tag in this series is [R2-2023-csp-2](https://github.com/folio-org/platform-complete/tree/R2-2023-csp-2). The versions of these modules included in that release are:

* ui-courses ([UICR-198](https://folio-org.atlassian.net/browse/UICR-198)) -- v6.0.3
* ui-ldp ([UILDP-146](https://folio-org.atlassian.net/browse/UILDP-146)) -- v2.0.1
* ui-plugin-eusage-reports ([UIPER-121](https://folio-org.atlassian.net/browse/UIPER-121)) -- v3.0.0
* ui-harvester-admin ([MODHAADM-80](https://folio-org.atlassian.net/browse/MODHAADM-80) **INCORRECT**) -- not included in flower releases, but the last release waa v2.1.0.
* mod-graphql ([MODGQL-180](https://folio-org.atlassian.net/browse/MODGQL-180)) -- v1.12.1
* mod-z3950 ([ZF-97](https://folio-org.atlassian.net/browse/ZF-97)) -- v3.3.5


## Stripes versions

The version of Stripes included in Quesnelia will be v9.1._x_ (for various incrementing values of _x_ as service patches are released), so we need to determine whether the last releases of these modules will work with Stripes v9.1._x_. (They all worked with Stripes v9.0.5, which was included in Poppy, but it's possible some of them were locked to v9.0._x_ series.)

In fact,
ui-courses v6.0.3,
ui-ldp v2.0.1
ui-plugin-eusage-reports v3.0.0
and
ui-harvester v2.1.0
all depend on Stripes ^9.0.0, so none of them need to be re-released in order to run on the new version of Stripes.


## Changes in modules

Since the most recent releases of the six modules:
* ui-courses has no substantive changes. Its package file has had its version number pre-emptively bumped to 6.1.0, but no code changes have been made. There have been some tweaks to the translations but these do not warrant a release.
* ui-ldp has an authentication fix (UILDP-145) in need of release.
* ui-plugin-eusage-reports has had no substantive changes: only a pre-emptive version bump and some translations.
* ui-harvester-admin has had no intrinsic changes since v2.1.0: only tweaks to the GitHub actions.
* mod-graphql has no changes at all since the most recent release, v1.12.1.
* mod-z3950 has had a release since the most recent Poppy, v3.4.0, but there have been no changes since then.

Conclusion: ui-ldp needs to be released, but nothing else does!

