<!--
date: 2022-02-05
link: https://archive.fosdem.org/2022/schedule/event/foreman_katello_leapp_elevate/
video: https://video.fosdem.org/2022/D.infra/foreman_katello_leapp_elevate.mp4
-->
# Migrating Foreman from EL7 to EL8 using LEAPP/ELevate

---

## `$ whoami`

Evgeni Golov

Senior Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

Note:
First of all, who am I? I'm a senior software engineer at Red Hat, working on the Foreman project and Red Hat Satellite, especially how to automate workflows around those.

---

# agenda

1. Why upgrading?
2. Different upgrade paths
3. LEAPP details

---

# Why upgrading?

* Newer is better (🙃)
* Foreman is [dropping EL7 support in 3.3](https://community.theforeman.org/t/deprecation-plans-for-foreman-on-el7-debian-10-and-ubuntu-18-04/25008)
* EL7 is slowly becoming a maintenance burden

Note:
* 3.3 is ~ Summer 2022
* e.g. no ostree for Katello
* Apache has bugs
* opens up capacity to look at EL9

---

# different upgrade paths

* on Debian: `sed 's/buster/bullseye/'`
  `apt update && apt upgrade`
* EL-based distributions don't support this
* classical answer "just redeploy" (backup/restore)
* you *can* make upgrades work, manually
* or use supported tooling like LEAPP/ELevate

---

## backup/restore?!

* not an *upgrade* per se
* you take a backup of your EL7 Foreman
* and restore it on EL8
* clean, supported, slow

Note:
* This is nice if you want to clean up the host
* Takes longer, especially if you use Katello and have a lot of content synced
* People hate *slow*

---

## manual upgrade

* add EL8 repos
* tinker with `yum`/`dnf` until all EL7 packages are replaced with their EL8 pendants
    * extra fun with SCL → Module "translation"
* dirty, unsupported, quick

Note:
* Architectural changes: SCLs to modules
* Differently named EL8 packages don't `Obsolete` the EL7 names
* "quick" only after you figured out what to do once…
* What if… we'd put this knowledge into code?

---

## LEAPP/ELevate

* LEAPP is the official Red Hat tool to do RHEL7 to RHEL8 upgrades
* ELevate is a project by AlmaLinux to extend support to AlmaLinux, CentOS, and other EL-derivatives
* contains logic how to map/update packages, services and configurations

Note:
* the logic is limited to base repositories
* it doesn't know about additional products/repos
* neither about SCLs and friends

---

## LEAPP/ELevate

* LEAPP can be extended with custom Actors
* users can write own Actors to extend support beyond the basics
* guess what we did…
* clean, supported, quick (🎉)

Note:
* Actors are essentially plugins
* You usually need multiple Actors, that get executed at different stages of the upgrade process

---

# LEAPP details

---

## Work in Progress

* Actors for the Satellite EL7 to EL8 upgrade are WIP: [PR#733](https://github.com/oamg/leapp-repository/pull/733)
* The code should equally work for Foreman/Katello 3.1+/4.3+

---

## Upgrade steps

* Replace all EL7 packages with EL8 versions
    * Replace SCLs with Modules
    * Translate package names
    * Remove unnecessary packages
* Update configuration files

---

## Modularity support

* Modules are a new concept in EL8, replacing SCLs
* Upgrading SCLs is out of scope for LEAPP
* Thus also no need for Modularity
* Support for enabling modules came recently in [PR#672](https://github.com/oamg/leapp-repository/pull/672)

Note:
* LEAPP needs module support for EL8 to EL9 upgrades
* Our Actor can translate the SCLs we use on EL7 to modules on EL8

---

## RPM and symlinks

* RPM can't replace a symlink with a directory ([BZ#2018131](https://bugzilla.redhat.com/show_bug.cgi?id=2018131))
* When you try to upgrade PostgreSQL 12 from SCL to module, exactly this happens
* DNF aborts the transaction
* Workaround: have a hook that removes that symlink before the DNF transaction starts

Note:
* Bad workaround: two transactions
* Good workaround: remove symlink via script
* Thankfully already known to LEAPP

---

## Our Installer is nice
## and slow

* instead of teaching LEAPP to translate configurations we run `foreman-installer`
* this is rather slow, but saves us from code duplication
* happens in "First Boot" phase

Note:
* Slow is ~5-10min for an empty Katello installation
* First Boot because we need DB and services running, and that doesn't happen in the intermediate initramfs

---

## LEAPP upgrade

```shell
# yum install leapp
# leapp preupgrade
# leapp upgrade
# reboot
```

Note:
* LEAPP requires metadata you get from Red Hat or Alma
* `preupgrade` will always find something wrong with your system on the first run
    * EL7 has a few default configs [pam, ssh], that won't work on EL8
* read the report and act accordingly

---

# Links

* [Foreman 3.3 drops EL7](https://community.theforeman.org/t/deprecation-plans-for-foreman-on-el7-debian-10-and-ubuntu-18-04/25008)
* [LEAPP project](https://leapp.readthedocs.io/en/latest/)
* [ELevate](https://almalinux.org/elevate)
* [ELevate quickstart guide](https://wiki.almalinux.org/elevate/ELevate-quickstart-guide.html)
* [Foreman/Katello/Satellite Actor PR for LEAPP](https://github.com/oamg/leapp-repository/pull/733)

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-twitter" aria-hidden="true"></i> [@zhenech](https://twitter.com/zhenech)

<i class="fa fa-comment" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)
