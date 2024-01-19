
# Maintaining over 40 Ansible modules: ~~3~~ 4 years later

---

## `$ whoami`

Evgeni Golov

Senior Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

---

## Foreman + Ansible = <span class="emoji">♥</span>

* Foreman has an API
* Everyone loves writing YAML instead of clicking in a GUI
* So we wrote modules, rewrote them again, refactored them and stuffed them into a collection
* This is the story of our journey

---

# Prelude: Motivation / WTF

---

## What's Foreman?

* lifecycle management tool for physical and virtual servers
* power management, provisioning, configuration
* Bare-Metal, VMware, RHV, OpenStack, GCE, Azure, etc
* huge plugin ecosystem (Katello, Monitoring, Ansible, …)

---

## What's Ansible?

* "radically simple IT automation engine"
* huge number of modules for various usecases
* writing own modules is very easy
* integrates well with REST APIs

---

## Why automating Foreman with Ansible?

* We have daily tasks in our environment
* WebUI and hammer don't scale well
* Using Ansible as a declarative API client

---

# Chapter 1: `ansible/ansible`

---

## `foreman` and `katello` modules in `ansible/ansible`

* Ansible upstream since 2.3 (2016)
* one module to rule them all, thus cumbersome to use
* uses the (Satellite specific) `nailgun` library
* mostly Katello oriented

Note:
* `nailgun` is written for Satellite
* `nailgun` branches only support what's in the matching Satellite release
* doesn't work well on non-Katello installations
* no support for Plugins outside of Satellite

---

## `foreman` and `katello` modules in `ansible/ansible`

* Turns out one maintainer for code in `ansible/ansible` is not enough
* Didn't have tests until 2018
* Deprecated since 2.8
* To be removed in 2.12

---

# Chapter 2: a new repository

---

## `foreman-ansible-modules.git`

* Started in June 2017
* A new repository under `@theforeman` organization
* Goal: central place for collaboration around Ansible modules for Foreman
* First step: split `foreman` and `katello` into "one module per entity"
    * started with 6 modules
* Centralized `module_utils`: July 2017

Note:
* user customizable `module_utils` came in Ansible 2.3
* before we could not have shared code when not living in `ansible/ansible`

---

# Chapter 3: Tests

---

## Chapter 3.1: test playbooks

* First set of tests added in November 2017
* Playbooks that would use the modules against a live server
* Good start, but expensive test execution
* Doesn't play well with Travis CI and friends

---

## Chapter 3.2: VCR based tests

* VCR (`vcrpy`) is a great way to record and replay HTTP requests/responses
* Allows recording "good" API interactions and replay them on Travis
* Added January 2018
* Ensured modules work on Python 2.7 + 3.5
* First `PlaybookCLI`, now `ansible-runner`
* Full coverage: August 2019

---

## Chapter 3.3: check mode tests

* All our modules support check mode
* We re-run the VCR based tests with `--check`

Note:
* Well, `katello_manifest` does not, but that is special anyways

---

## Chapter 3.4: sanity tests

* Ansible provides `ansible-test` for in-tree modules
* Since Ansible 2.9 it can also handle Collections
* We run `ansible-test sanity --venv plugins/` across all supported Pythons

---

## Chapter 3.6: expected change tests

* Our test playbooks execute every task twice
* The first execution is expected to have `changed=True`
* The second `changed=False`
* This ensures the modules are idempotent

---

## Chapter 3.7: diff mode tests

* Our modules return before/after `diff` data to Ansible
* We access that data in our test playbooks and analyze the content

---

# Chapter 4: Documentation

---

## Chapter 4.1: building documentation

* All modules have `DOCUMENTATION` populated
* We use `build-ansible.py document-plugins` with a customized template
* Ansible internal, our use of it breaks sometimes
    * Would be great to have official tooling
    * Automatic builds on Galaxy?
* Need to figure out how to autopublish docs

---

## Chapter 4.2: documentation fragments

* Ansible 2.8 introduced documentation fragments
* We use them heavily to document common parameters (credentials etc)
* Fragments for return values would be cool

---

# Chapter 5: ForemanAnsibleModule

---

## Chapter 5.1: ForemanAnsibleModule

* `ForemanAnsibleModule` is a sub-class of `AnsibleModule`
* Simplified definition of common parameters in `argument_spec`
* Import error handling
* Entity create/update/delete/compare helpers
* before/after `diff` handling

Note:
* February 2019
* `82a8ab4d31aec0cfd7eebfa8fe76e07532f52a6e`

---

## Chapter 5.2: ForemanEntity…, KatelloAnsibleModule, …

* Further sub-classing useful
* ForemanEntityAnsibleModule adds a `state` parameter
* KatelloAnsibleModule makes makes `organization` required

---

# Chapter 6: libraries

---

## Chapter 6.1: nailgun

* We started with the `nailgun` library
* Originally developed by Satellite QE
* Targeted at Satellite environments
    * no support for non-Satellite plugins
    * released at the same cadence as Satellite
* Designed to test the Satellite API

Note:
* Doesn't use `apidoc.json`
* Re-implements
    * Validation
    * Resource definition

---

## Chapter 6.2: apypie

* `nailgun` was fine when we targeted Satellite environments
* Katello (and Foreman) were moving quicker
* Decided to write an own API library
    * using the published `apidoc.json`, thus mostly version agnostic
* Switching libraries was rather easy due to the abstraction we've built
    * And tests, tests will save you!

Note:
* Still took 7 months from `7369ad3d860f715a5a6c110116ea501b10beb74c` to  `79b27565788c7e3ccc117782ff1ba78e7f5192ed`

---

# Chapter 7: use the force

---

## Chapter 7.1: use the force of the argument_spec

* Ansible supports complex (nested) `argument_spec`s
    * `elements='dict', options=dict(…)`
    * Allows better checking of complex parameters

---

## Chapter 7.2: use the force of the entity_spec

* We always had a need to map from Ansible param names to Foreman API parameters
* This resulted in the introduction of the `entity_spec`
    * `argument_spec` extended with Foreman specific data
    * The plain `argument_spec` can be generated from it

Note:
* Mostly to map things like `organization` to `organization_id`

---

## Chapter 7.3: use the force of the entity_spec pt. 2

* Many modules perform simple CRUD operations:
    * take user input
    * find matching entity
    * create/update/delete based on input
    * report
* We used to have write *code* for that, now this is generated from the `entity_spec`

Note:
* `747f59de4594d90900557752148366ab27c87afb`

---

# Chapter 8: Community

---

## Chapter 8: Community

* Originally started as "my team needs this"
* Quickly gained contributions from ATIX
* Today: 35 contributors, many from Red Hat and ATIX
* Developers, Consultants, Ops, Customers
* Adding their usecases and features

---

## Chapter 8: Community

* Initial contribution was hard, duplicated code, hard to test
* Increased contribution when we moved to a more centralized codebase
* Having a collection and RPMs made consumption easier
* Recording VCR test results is still the biggest blocker

---

# Chapter 9: Outlook

---

## Chapter 9: Outlook

* `foreman_host` supporting *ALL* the parameters
* official roles to support central workflows
* *more* modules
* documentation autopublishing and versioning
* easier contribution
* collection defaults like [module defaults groups](https://docs.ansible.com/ansible/devel/user_guide/playbooks_module_defaults.html#module-defaults-groups)?

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-twitter" aria-hidden="true"></i> [@zhenech](https://twitter.com/zhenech)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)
