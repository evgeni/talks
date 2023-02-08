# Maintaining over 70 Ansible modules: <br/>7 years later

---

## `$ whoami`

Evgeni Golov

Principal Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

---

Three years ago, this talk was called  
[**Maintaining over 40 Ansible modules:  
4 years later**](https://cfp.cfgmgmtcamp.org/2020/talk/U7CGMZ/).  
Let's see what the three years brought us,  
*besides* 30 new modules!

Note:
* No, I did not only do a `s/4/7/g`

---

1. What happened in Foreman in regard to Ansible?
2. What happened in the Ansible community overall?
3. What we're looking for in the *next* three years?

Note:
* Well, maybe not overall, but that affects us?

---

# Foreman

---

## Releases

* Three major releases 🥳:
    * 1.0.0 (2020)
    * 2.0.0 (2021)
    * 3.0.0 (2021)
* 27 releases in total (since 0.4.0)
* Only one "out of order" bugfix release (1.5.1)

Note:
* 1.0 was the big rename and polishing release
* 2.0 "just" broke role variables by adding a common prefix
* 3.0 dropped Ansible 2.8 support and changed inventory default to report

---

## New Contributors

* 50 new contributors!
* 2019: 35, 2023: 85!

Note:
* I guess we delivered on the "make contribution easy" promise?
* But I still think it's harder than it should be

---

## New Modules

* Can't avoid talking about these!
* 15 `*_info` modules to fetch information about individual resources

---

## New Modules

* `content_export_*`
* `discovery_rule`
* `puppetclasses_import`
* `job_invocation`
* `smart_proxy`
* `http_proxy`

Note:
* `content_export_(library|repository|version|info)`
* Import ones are still missing
* content: Jeremy, discovery: Jeffrey, invocation: Peter Ondrejka, smart: James Stuart

---

## New Roles

* We promised those last time!
* 21 in total
* most around repetitive tasks ("create the following list of resources")

---

## New Roles

* `content_rhel` sets up everything you need to serve RHEL machines with content
* `convert2rhel` sets up everything you need to convert CentOS and Oracle Linux machines to RHEL
* `content_view_version_cleanup` removes old and unused Content View Versions (`cvmanager` anyone?)

---

## New Collections

* `theforeman.operations` takes care of operational tasks like installing Foreman, configuring backup, etc
* `redhat.satellite` is the branded and supported version of `theforeman.foreman` for Satellite customers
* `redhat.satellite_operations` is the branded and supported version of `theforeman.operations` for Satellite customers

---

## New Collections

* `pulp.squeezer` allows managing a Pulp3 instance

Note:
* The only reason I am in this room!

---

## Execution Environment

* We provide metadata to build an EE
* Makes it easy to run the code in an controlled environment

---

## packages

```console
# {apt|dnf} install ansible-collection-theforeman-foreman
# {apt|dnf} install ansible-collection-theforeman-operations
```

---

## Vendored apypie

* We originally wrote `apypie` as a standalone project
* Installing from Ansible Galaxy meant `apypie` was not installed
* We're now vendoring `apypie` to make installations easier

Note:
* Ansible doesn't allow to specify dependencies on anything but other collections
* This is fine for packages, we can express things there
* `bindep.txt` is only used for Execution Environments

---

## live tests

* Turns out if you *only* test against recorded API responses, you miss changes
* Added easy way to run tests in "live" mode (without recording)

Note:
* Not run by default in CI, but e.g. Satellite QE uses that mode

---

# Ansible

---

## Automatic documentation publishing

* [`antsibull-docs`](https://github.com/ansible-community/antsibull-docs) extracts documentation
* we have simple GitHub workflow that runs this on every PR merge and release
* (versioned) results visible on https://theforeman.org/plugins/foreman-ansible-modules/
* Galaxy still does not show docs, but Automation Hub does

Note:
* much more stable than our old hacks ;-)

---

## Semi-Automatic changelog generation

* Ansible uses changelog fragments for writing changelogs
* Collected by [`antsibull-changelog`](https://github.com/ansible-community/antsibull-changelog)
* Automatically detects new modules

Note:
* Releasing is: edit version, run `antsibull-changelog release`, commit, tag, push

---

## Module defaults groups

* Supported since ansible-core 2.12 for collections
* Static list in `meta/runtime.yml`
* We scripted the generation

```yaml=
- hosts: localhost
  module_defaults:
    group/theforeman.foreman.foreman:
      server_url: https://foreman.example.com
      username: admin
      password: changeme
  tasks: …
```

Note:
* In the collection since 3.4.0

---

## ansible-lint only-builtins

* In `ansible-lint` since 6.1
* Useful to verify your playbooks are not using modules outside of `ansible-core`
* We lint our roles to avoid surprise dependencies

```yaml=
---
enable_list:
  - only-builtins
only_builtins_allow_collections:
  - theforeman.foreman
```

---

## galaxy-importer testing

* Ansible Galaxy and Automation Hub use `galaxy-importer` for importing of Collections
* We run it as part of CI to ensure smooth releases

```ini=
[galaxy-importer]
CHECK_REQUIRED_TAGS = True
```

Note:
* That tag requirement only exists on AH, not Galaxy

---

# Outlook

---

## Content View Filters

* The existing `content_view_filter` module is
    * confusing
    * buggy
    * missing features
* New `content_view_filter_rule` module for managing the Rules independently
* New `*_info` modules for Filters and Rules

---

## CDN configuration

* Katello supports different ways to obtain Red Hat content
* We don't support configuring those
* But that's useful for disconnected setups that use content exports

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-twitter" aria-hidden="true"></i> [@zhenech](https://twitter.com/zhenech)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)

Note:

The [Foreman](https://theforeman.org) community maintains a [collection of over 70 Ansible modules](https://github.com/theforeman/foreman-ansible-modules) for interaction with the Foreman API and the various plugin APIs. At [cfgmgmtcamp 2020 we talked about the first four years of that journey](https://cfp.cfgmgmtcamp.org/2020/talk/U7CGMZ/) and promised you quite a lot for the future.

Today we want to talk what those three years allowed us to achieve, how priorities shifted and what else happened.
Including:
* Three major releases: 1.0.0 (2020), 2.0.0 (2021), 3.0.0 (2021) 🥳
* 27 releases in total (since 0.4.0)
* Over 1.2 million downloads on Galaxy
* 21 new roles
* `redhat.satellite` collection
* `theforeman.operations` collection

And of course we will also talk about what we think is next!

Old outlook:
* `foreman_host` supporting *ALL* the parameters
* [x] official roles to support central workflows
* [x] *more* modules
* [x] documentation autopublishing and versioning
* easier contribution
* [x] collection defaults like [module defaults groups](https://docs.ansible.com/ansible/devel/user_guide/playbooks_module_defaults.html#module-defaults-groups)?
