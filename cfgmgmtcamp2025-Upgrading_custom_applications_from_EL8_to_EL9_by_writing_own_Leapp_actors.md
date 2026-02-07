<!--
date: 2025-02-03
event: cfgmgmtcamp 2025
link: https://cfp.cfgmgmtcamp.org/ghent2025/talk/XH77AU/
video: https://www.youtube.com/watch?v=c5T8qxgWnAA
-->
# Upgrading custom applications from EL8 to EL9 by writing own Leapp actors

<!--

To upgrade the operating system underneath an application, everybody should just redeploy said application on a new system, which thanks to automation is both easy and fast.

After recovering from the shock of reading "just", "easy" and "fast" in once sentence, we have to realize that a fresh deployment is not always the easiest/fastest path forward, or maybe not even possible at all. This is where distributions come to help us by offering support for major upgrades "in place".

For Enterprise Linux such upgrades are done by [Leapp](https://leapp.readthedocs.io/), which is both a framework to orchestrate complex upgrades and a collection of helpers (so called actors) for upgrading Enterprise Linux setups with common applications installed.

However, "common applications" might not include the one *you* are developing and have deployed on-premises at many customers.

In this talk we will show how we developed the custom actors required for upgrading [Foreman](https://theforeman.org) from EL8 to EL9, which challenges we faced and which shortcuts we took.

-->

---

## `$ whoami`

Evgeni Golov

Principal Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

---

> We don't run those expensive servers to run Linux on them,
> we run them to run an application ontop.

Note:
* A customer once told me that
* It struck, as it's right -- the OS itself is rather pointless on its own
* Nevertheless the OS is super important for HW, basic libs etc
* And every now and then we need to upgrade to the next OS version

---

# Just redeploy!

- <!-- {_class="fragment"} --> works well for state-less
- <!-- {_class="fragment"} --> works well for single-state
- <!-- {_class="fragment"} --> doesn't work when state is all over the place

Note:
* If that works well for you -- perfect, don't bother listening to the rest of the talk
* But let's see when it can actually work well
* state-less: reverse proxy, HAProxy, nginx, "worker" node…
* single-state: PostgreSQL where everything is in `/var/lib/pgsql`
* Sadly, Foreman is in that last bucket
* PostgreSQL, Pulp, TFTP, DHCP, Ansible collections, Puppet modules, SCAP reports, etc etc
* Luckily no more MongoDB
* We can always redeploy based on a full backup/restore run, but that takes time

---

# Just dist-upgrade!

- <!-- {_class="fragment"} --> Debian: edit sources && <code>apt upgrade</code>
- <!-- {_class="fragment"} --> Fedora: <code>dnf system-upgrade</code>
- <!-- {_class="fragment"} --> EL: <code>leapp upgrade</code>

Note:
- Debian upgrades the system while it's running
- Fedora and EL "reboots into a special environment" to do the upgrade
- Leapp includes code and data to allow jumping 5-10 years of OS development in one go
- It works under the assumption that this jump cannot be expressed by package relationship and postinst scripts alone
- It also allows to codify checks of the form "your setup won't be supported in the next release"
- Ubuntus `do-release-upgrade` does something similar
- But this won't help us just yet…

---

# Just the OS

While the way the various tools do the upgrade vary, they have one thing in common:

they can only really update the main operating system

Note:
- Do you hate the word "just" yet? You should!
- No matter how smart the tooling is, it doesn't know about *YOUR* application

---

# Just the OS

- <!-- {_class="fragment"} --> Testing only happens for upgrades of the OS
- <!-- {_class="fragment"} --> If the application is packaged it might be removed due to unmet dependencies
- <!-- {_class="fragment"} --> If the application is unpackaged it might not work due to unmet dependencies

Note:
- Sure, if your app is part of the OS (bind, postfix, nginx), you're good. But when is it?
- No idea what's worse, removed or not working
- Probably not working, as detecting that can be more complicated?

---

# Extending the upgrade to include the application

Note:
* Maybe you're lucky and you only need to rebuild the app and install the new packages
* I wasn't that lucky, but at least you get to hear an interesting talk? :)

---

## Prerequisites

- the OS itself is upgradeable
- the application can actually run on the new OS

Note:
- This *should* be obvious
- So your first step is to make the app run
- While doing so, take notes what differs on the new OS
- But also, try to avoid a few things!

---

## Things to avoid

<span class="emoji fragment">change 🙃</span>

- <!-- {_class="fragment"} --> PostgreSQL upgrades
- <!-- {_class="fragment"} --> Puppet upgrades
- <!-- {_class="fragment"} --> other non-critical upgrades

Note:
* Whether any of these change, we learn when we try do freshly deploy the app on the new OS
* We should try to align "stack versions" outside of the OS upgrade if possible
* For example in Foreman, we upgraded to PostgreSQL 13 to align with EL9 while upgrading to 3.11
* And then during the lifetime of 3.11/3.12 allowed the jump to EL9
* Slide 10 and we still didn't look at how leapp works, shall we?

---

## leapp

<div class="mermaid">
  <pre>
    flowchart LR
      analyze --> report;
      report -- problems found --> analyze;
      report --> upgrade;
      upgrade;
  </pre>
</div>

Note:
* Each phase can be enhanced by adding own code, called actors
* Internally there are many more phases, but this is the idea
* For us the details of the upgrade phase are important

---

## leapp

<div class="mermaid">
  <pre>
    flowchart LR
      analyze --> report;
      report -- problems found --> analyze;
      report --> upgrade;
      subgraph upgrade;
        prepare --> run:::initrd;
        run --> firstboot:::boot;
      end;
      classDef initrd fill:#f96;
      classDef boot fill:#096;
  </pre>
</div>

Note:
* The different parts of the upgrade are important, as you can do different things there
* "prepare" creates a live environment, minimal changes to the existing system, revertable
* "run" performs the RPM replacement etc, point of no return
* "firstboot" is the first boot into the new systems, finalizing things
* Let's see where we had to hook in to upgrade Foreman

---

## `satellite_upgrade_facts`

```python[|1|2-3|4]
consumes = (InstalledRPM, UsedRepositories)
produces = (RepositoriesSetupTasks, RpmTransactionTasks,
            SatelliteFacts)
tags = (IPUWorkflowTag, FactsPhaseTag)
```

Note:
* This is the "analyze" step
* `consumes` denotes which information we need to function
* it also influences the ordering of the actors
* `produces` is what others can consume (and ordering!)
* `tags` define which phase we run in
* `FactsPhaseTag`?! I told you there are more phases!

---

## `satellite_upgrade_facts`

* checks whether `foreman` or `foreman-proxy` is in `InstalledRPM`
* collects information about the system (installed plugins, PostgreSQL, etc) and produces `SatelliteFacts`
* creates a list of packages to install/replace/remove and produces `RpmTransactionTasks`
* (Satellite-only) reconfigures RHSM repositories using `RepositoriesSetupTasks`

Note:
* This is the "analyze" step
* On systems w/o Foreman nothing is produced
* `SatelliteFacts`?! Next slide!
* All later actors can take the absence of `SatelliteFacts` as "no action needed"

---

## `SatelliteFacts`

* Leapp calls this a `Model`
* Essentially a data-only class
* Can be produced and consumed by actors
* The only way actors can "communicate" with each other
* We store information about plugins and PostgreSQL setup there

---

## `satellite_upgrade_ services`

```python[|1|2|3]
consumes = (SatelliteFacts,)
produces = (SystemdServicesTasks,)
tags = (IPUWorkflowTag, FactsPhaseTag)
```

Note:
* This is the still "analyze" step
* Just consuming the previously created Facts and generating some others

---

## `satellite_upgrade_ services`

* We need to perform migration tasks without services running
* We tell Leapp to disable them

Note:
* This is the still "analyze" step
* Technically we could do this in `upgrade_facts`
* In the past there was no `SystemdServicesTasks` so we had to delete symlinks on our own -- in the "run" step
* And that only could happen in a different phase ;)

---

## `satellite_upgrade_check`

```python[|1|2|3]
consumes = (SatelliteFacts,)
produces = (Report,)
tags = (IPUWorkflowTag, ChecksPhaseTag)
```

Note:
* This is the "report" step

---

## `satellite_upgrade_check`

* Only was needed for 7to8
* Report whether PostgreSQL could be migrated automatically or user had to intervene

Note:
* This is the "report" step
* Remember I said we should avoid touching PostgreSQL?

---

## `satellite_upgrade_data_migration`

```python[|1|2|3]
consumes = (SatelliteFacts,)
produces = ()
tags = (IPUWorkflowTag, ApplicationsPhaseTag)
```

Note:
* `ApplicationsPhaseTag` is part of the "main" upgrade that happens in a live system - "run" in the graph

---

## `satellite_upgrade_data_migration`

* Only was needed for 7to8
* Moves PostgreSQL data to the new location on EL8

Note:
* `ApplicationsPhaseTag` is part of the "main" upgrade that happens in a live system - "run" in the graph
* Runs after the RPMs were upgraded
* This part was super error prone :/

---

## `satellite_upgrader`

```python
consumes = (SatelliteFacts, )
produces = ()
tags = (IPUWorkflowTag, FirstBootPhaseTag)
```

Note:
* This is "firstboot", as we need things to start up, which doesn't work in "run"

---

## `satellite_upgrader`

* Reindex the database
* Run the installer

Note:
* This is "firstboot", as we need things to start up, which doesn't work in "run"
* Reindex is required because PostgreSQL relies on locale data from the OS and that changed
* Installer should rectify all configuration changes that are needed on the new OS

---

## Actors summary

* 5 actors
* 2 7to8 specific
* 1 could be refactored into another

---

## Shortcuts

* Installer takes care of the configuration
* We don't need to migrate any data, we took care of that before
* We don't need to replace any packages

Note:

Otherwise:
* leapp needs to handle config file changes
* leapp needs to handle data migration
* leapp needs to handle package replacement -- we could use PES

---

## Learnings

* Doings this a second time helps
* If your actor is version-agnostic, put it in `common`

---

# Links

* [Inplace Upgrade Workflow](https://leapp.readthedocs.io/en/latest/inplace-upgrade-workflow.html)
* [Foreman/Satellite el8toel9 upgrade PR](https://github.com/oamg/leapp-repository/pull/1181)

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)
