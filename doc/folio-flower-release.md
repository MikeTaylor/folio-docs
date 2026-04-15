# Mike's module releases for FOLIO Trillium


## Background

General instructions can be found in
[Preparing for a new FOLIO release](https://github.com/folio-org/ui-ldp/blob/master/doc/new-folio-release.md)


## Modules

I am responsible for eight modules which may need releasing. The previous release was Sunflower (R1-2025), and the most recent platform-complete tag in this series is [R1-2025-okapi-csp-5-1](https://github.com/folio-org/platform-complete/tree/R1-2025-okapi-csp-5-1). The versions of these modules included in that release are:

 * **DONE** ui-courses ([UICR-240](https://folio-org.atlassian.net/browse/UICR-240)) -- v7.0.3
 * **DONE** ui-ldp ([UILDP-195](https://folio-org.atlassian.net/browse/UILDP-195) -- v3.1.2
 * **DONE** ui-plugin-eusage-reports ([UIPER-157](https://folio-org.atlassian.net/browse/UIPER-157)) -- v4.0.1
 * **WONTDO** ui-harvester-admin ([UIHAADM-162](https://folio-org.atlassian.net/browse/UIHAADM-162)) -- **not included in flower releases** but most recent release is v3.0.0.
 * ui-inventory-import ([UIINIMP-34](https://folio-org.atlassian.net/browse/UIINIMP-34)) -- N/A
 * mod-graphql ([MODGQL-202](https://folio-org.atlassian.net/browse/MODGQL-202)) -- v1.13.1
 * mod-z3950 ([ZF-126](https://folio-org.atlassian.net/browse/ZF-126)) -- v4.0.1
 * **DONE** mod-reporting ([MODREP-51](https://folio-org.atlassian.net/browse/MODREP-51)) -- v1.5.0


## Stripes versions

According to [the Offically Supported Technologies page for Trillium](https://folio-org.atlassian.net/wiki/spaces/TC/pages/730891059/Trillium), the version of Stripes included in Trillium will be v10.1._x_ (for various incrementing values of _x_ as service patches are released), and we will use React v18.2._x_ (but nothing is said about other peer-dependencies such as `react-intl` and `react-router`).

So we need to determine whether the last releases of these modules will work with Stripes v10.1._x_. (They all worked with Stripes v10.0.x, which was included in Sunflower, but it's possible some of them were locked to v10.0._x_ series). React 18.2 was also used in Sunflower, so we can be confident about it.


## Changes in modules

Since the most recent releases of the six modules:
* ui-courses has no substantive changes. Its package file has had its version number pre-emptively bumped to 6.1.0, but no code changes have been made. There have been some tweaks to the translations but these do not warrant a release.
* ui-ldp has an authentication fix (UILDP-145) in need of release.
* ui-plugin-eusage-reports has had no substantive changes: only a pre-emptive version bump and some translations.
* ui-harvester-admin has had no intrinsic changes since v2.1.0: only tweaks to the GitHub actions.
* mod-graphql has no changes at all since the most recent release, v1.12.1.
* mod-z3950 has had a release since the most recent Sunflower, v3.4.0, but there have been no changes since then.

Conclusion: ui-ldp needs to be released, but nothing else does!

