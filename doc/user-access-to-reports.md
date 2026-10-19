# Reporting app: restricting user access to reports

<!-- md2toc -l 2 user-access-to-reports.md -->
* [Introduction](#introduction)
    * [The situation today](#the-situation-today)
    * [The problem](#the-problem)
    * [How the solution will look](#how-the-solution-will-look)
* [Representation of user permissions](#representation-of-user-permissions)
* [Candidate implementations of restrictions](#candidate-implementations-of-restrictions)
    * [UI-only restrictions](#ui-only-restrictions)
    * [Restrictions implemented in mod-reporting](#restrictions-implemented-in-mod-reporting)
    * [Restrictions implemented in Metadb](#restrictions-implemented-in-metadb)
* [Recommendation](#recommendation)


## Introduction


### The situation today

In [FOLIO's Reporting app](https://indexdata-test.folio.indexdata.com/ldp), the [**Run report** tab](https://indexdata-test.folio.indexdata.com/ldp/templated) in the left pane provides a set of pre-authored reports that the user can run by filling in values for parameters that are configured as part of the report definition.

Reports are sourced from repositories in publicly visible version-control systems -- currently GitHub and GitLab, though others may be supported in future. The set of available report repositories is maintained in the settings page, [**Settings** &rarr; **Reporting** &rarr; **Report repositories**](https://indexdata-test.folio.indexdata.com/settings/ldp/tqrepos). Each repository is specified by five or six parameters: a type (GitHub or GitLab), a base URL (for GitLab repositories only), and a version-control user, repository name, branch name and directory.


### The problem

At present, every report is available for any user to run. This may not be appropriate for many reasons. For example: some reports may provide information that the user should be be allowed to see; others may take a long time to run and should only be invoked by those who understand the implications.


### How the solution will look

We want the ability to assign a set of one or more users to a given report repository, and allow only those users to run the reports therein. [UILDP-199](https://folio-org.atlassian.net/browse/UILDP-199) has been filed to capture this requirement. The idea is that in the settings page, each report repository will have an additional field listing the users who are allowed to run reports from that repository. Management of the list will follow the UI template laid down for access to acquisition units in [the Aquisitions units app](https://indexdata-test.folio.indexdata.com/settings/acquisition-units/a3496137-8d2b-4d18-bd4b-e8fca05fec75/view): at the bottom of each report repository's definition, there will be an **Assign users** button that pops up a user selection plugin.

But how to implement this? There are two major technical decision to be made:
* How to represent the availability of a report repo to a user?
* Where in the software stack to implement the restrictions?



## Representation of user permissions

Because assigning a user to a report repository is, in a broad sense, giving that user a permission, it's an appealing idea to implement the permissions to access report repository as FOLIO permissions. This would be done by providing a FOLIO permission for access to report repo, and assigning that permission to users either by the existing permission management facilities or via the **Assign users** button described above.

Since the set of report repos is dynamic, this cannot be achieved using a static set of permissions. But could it be done using Okapi's `permissionsDesired` facility, using wildcards [as in the mod-settings module descriptor](https://github.com/folio-org/mod-settings/blob/5f1b31a59e642955df4f1fa638656066280f79b1/descriptors/ModuleDescriptor-template.json#L147-L159)?

This is probably possible, but cumbersome in at least two ways.

1. The permissions would need to be named in a way that specifies which report repository they refer to. But since report repos are defined by a set of five or (for GitLab repos) six fields, these fields would all need to be munged into a single symbol to be used as the permission-name component. We would then have permission names like `mod-reporting.report-repo.github----senyl--test_reports--main--circulation.read`. Here we use a pair of hyphens to separate the report-repo configuration elements type, base URL (empty in this example), user, name, branch and directory.

2. Even if we were prepared to put up with such permission names, they would be sensitive to changes in any part of the report-repo definition. If for example the directory-name changed, then all users who had permission to use the repo would lose it, as their permission-names would no longer match.

Once we also consider the uncertainties of permission management under Eureka, this approach seems to introduce more complexity and doubt than it's worth.

Instead, we will simply store the list of which users are allowed to use which report repos in the same settings object that already contrains the report-repo configuration -- adding a seventh field which is the list of authorized users. (This does not introduce any new security loopholes, as any user with authorization to modify the user permissions for a report repo already has authorization to change the definition of the report repo itself.)

Extending the **Report repositories** settings page to allow management of authorized users is its own UI-only task, separate from that of enforcing restrictions.



## Candidate implementations of restrictions

How to enforce users' per-repo permissions? Enforcement would be easy to implement in the UI, but of course easy to bypass. That would have some value as a low guardrail helping to keep people away from reports they’re not meant to be using, but not at all as a razor-wire fence to keep out people determined to do so. Implementing such restrictions on the server side, in mod-reporting, is much more secure but correspondingly more work. And wiring the notion of per-user permissions right though to individual users at the Metadb level would be more secure still, but a lot more work, including designing elements of the solution that we don’t have a handle on yet.

Here we consider the three implementation approaches from the simplest (and least secure) to most complex (and most secure).


### UI-only restrictions

This is the most lightweight way to implement per-user restrictions is in the UI: the Reporting app would not list reports that are not available to the logged-in user, and if the user hacked the URL to land on the page of such a report it would refuse to display it or send it to mod-reporting.

This solution would be of some value, but would be very easy for a user to bypass: it is a guard against error, but not against malice. A user could easily formulate a WSAPI request against mod-reporting that asks it to run an unauthorized report.

If there are other clients that invoke reports via mod-reporting, such as a daily batch job that uses `curl` or similar, such clients would log in as a specific FOLIO user. This solution would do nothing to prevent such processes from accessing reports that ought not to be accessible to the FOLIO user that the process runs as.


### Restrictions implemented in mod-reporting

A further layer of security can be introduced by having mod-reporting verify for itself, on the server side, that when a user ask to run a report, that user is authorized for the report repo it's taken from.

This would require that mod-reporting pick apart the path of the requested report to determine what repo it's from, then load that repo's definition from mod-settings, and check whether the logged-in user is among those listed as authorized to run reports from that repo. Since the mod-reporting code already accesses mod-settings (to discover the location of the reporting database), this may not be especially onerous.

The trickiest part of this will be resolving a report URL, such as `https://github.com/seneyl/test_reports/blob/main/circulation2/additional_directory/circ_by_group.sql`, into a report-repo definition: in this case, type=`github`, user=`seneyl`, repo=`test_reports`, branch=`main`, directory=`circulation2/additional_directory`. There is some potential ambiguity in the case that multiple report repos are defined, one targetting a subdirectory of another. In such cases, we can either resolve on the most specific of the nesed report repos, or allow access if the user is authorized for any of the repos.

If we implement restrictions in mod-reporting, malicious users will be unable to gain access to unauthorized reports by hand-crafting WSAPI requests: the module will reject (and probably log) such requests.



### Restrictions implemented in Metadb

A yet more secure option would be to have Metadb itself enforce access to reports. This would offer two advantages:

1. Accesses to Metadb from sources other than mod-reporting could be subject to gatekeeping.
2. Metadb could in principle enforce column-level access restrictions.

Exactly how this would work, though, is not clear:

* Metadb has its own system of users, not related to FOLIO users. Some kind of synchronization would be required.
* Metadb at this point has no notion of a report repo -- only of fetching a single report from a specified URL. This notion would need to be added.
* Metadb has no obvious way to access the list of which users are authorized for a given report repo: it cannot in general reach back into the FOLIO instance that invoked it. The obvious solution is to have mod-reporting send the relevant information to Metadb, but that simply makes mod-reporting the gatekeeper of security. Metadb could in principle pluck this information out of the mod-settings data that it harvests, but as well as being conceptually impure, this would introduce synchronization difficulties.

In light of these problems, it's worth revisiting and re-evaluating the advantages noted above. Regarding the first, Metadb instances are typically firewalled and can only be accessed from trusted IP addresses, so protecting against requests from sources other than mod-reporting may be of only theoretical interest. On the second, [UILDP-199](https://folio-org.atlassian.net/browse/UILDP-199) includes the observation:

> Granting access to tables based on FOLIO permissions may be too restrictive. Some users may need access to a data point outside of the scope of their FOLIO permissions.  
> Example: Acquisitions may need data about the patron group, but do not have permissions in the Users app. The report author could include this column in the query to provide them with the data they need without them having access to patron-specific details.  

In which case having Metadb enforce column-level access restrictions might be actively harmful.



## Recommendation

UI-only enforcement is probably insufficient; Metadb-level enforcement is complicated, and requires significant changes to existing models plus some uncertainty.

So **enforcement at the level of mod-reporting is the way to go**.



