# Maintaining over 80 Ansible modules: <br/>8 years later

---

## `$ whoami`

Evgeni Golov

Principal Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

---

Four years ago, this talk was called  
[**Maintaining over 40 Ansible modules:  
4 years later**](https://cfp.cfgmgmtcamp.org/2020/talk/U7CGMZ/).  
Last year, this talk was called  
[**Maintaining over 70 Ansible modules:  
7 years later**](https://cfp.cfgmgmtcamp.org/2023/talk/SSAXCF/).  

With only one year to cover, this should be a short one?

Note:
* No, we do not add exactly so many modules to make the years line up with the number of modules.

---

1. What happened in Foreman in regard to Ansible?
2. What happened in the Ansible community overall?
3. What we're looking for in the *next* years?

Note:
* Well, maybe not overall, but that affects us?

---

# Foreman

---

## Releases

* One major release 🥳:
    * 4.0.0 (2024)
* 8 releases in total (since 3.8.0)
* No "out of order" bugfix releases
* Almost 2 million downloads on Galaxy

Note:
* 3.8.0 was latest at cfgmgmtcamp2023
* 4.0.0 breaking changes:
  * `content_view_filter` doesn't manage rules anymore
  * no more `localhost:3000` default for the inventory

---

## New Contributors

* 16 new contributors!
* 2019: 35, 2023: 85, 2024: 101!

Note:
* In 2023 we had 50 new for a 3 year span, so 16.6 per year, quite consistent!
* Greg, where are my graphs?!

---

## New Modules

* Can't avoid talking about these!
* 8 new modules in total

---

## New Modules

* `content_view_filter_info`
* `content_view_filter_rule_info`
* `content_view_filter_rule`

Note:
* `content_view_filter_*` by Paul - not net new, but huge amount of work done
* This was promised last year, yay delivered!

---

## New Modules

* `hostgroup_info`
* `registration_command`
* `smart_class_parameter_override_value`
* `wait_for_task`
* `webhook`

Note:
* `hostgroup_info` by Louis - new contributor
* `wait_for_task` by Julien Godin from camptocamp.com - new contributor
* `webhook` by Griffin - new contributor

---

## New Roles

* One new: `locations`

---

## New Collections

None -- we think we're settled (for now?)

Note:
* There are several collections/roles building upon ours, adding more complex tasks
* They are solving the 1% problems, while we aim for the other 99% common cases

---

## live tests

* Last year we made it easy to run tests "live"
* But we didn't ship them in the release artifact
* Now we do and everyone can verify things in their environment

Note:
* Otherwise it's hard to obtain the "right" set of tests without poking around Git and tags etc

---

## live deep dive session

* Deep Dive about contributing to our collection
* https://www.youtube.com/watch?v=PdbGMdxSeRA
* You really should follow [@Foreman](https://www.youtube.com/@Foreman) anyway

Note:
* Originally asked by the support org at Red Hat

---

## Strict var naming enforcement in our roles

* ansible-lint has rules for [variable naming](https://ansible.readthedocs.io/projects/lint/rules/var-naming/)
* For roles, all variables should start with the role name as kind of a namespace
* This doesn't work well for collections
* So we set `var_naming_pattern: "^(__)?foreman_[a-z_][a-z0-9_]*$"` now

---

## diff_mode and check_mode support documentation

* Our modules "always" properly supported diff mode and check mode
* Documentation was added in 3.10.0 so that `ansible-doc` and web docs properly show this fact

---

## no_log in roles

* Arguments of modules that carry secrets are tagged `no_log: true`
* There is no matching concept for roles
* So when our roles loop over a set of data, they would log possibly secret information
* One can pass `no_log: true` when looping, hiding *all* data

Note:
* We often do `no_log: "{{ secret is defined }}"` so it only hides when the secret param is actually defined

---

## OpenStack support in `compute_resource`

* As different Compute Resources need different parameters, the module needs to know that
* The mapping for OpenStack was missing
* But luckily a community member contributed the fix

---

## Creating "real" releases on GitHub

* We used to just tag and trigger a release to Galaxy
* Galaxy doesn't show the changelog
* Galaxy the only place to hold the release artifact
* Quite easy to extend the release workflow to also create a GH release, attach the tarball and post the changelog

Note:
* Release artifact can be recreated from the git tag, but it won't be bit-identical
* Releases to Automation Hub are not automated and we messed up at least once
* `antsibull-changelog generate --output release-changelog.txt --only-latest`

---

## Minimum supported Ansible version 2.9.17

* `requires_ansible: '>=2.9'` needs to be `X.Y.Z`
* 2.9.0 does not support collections properly
* roles inside collections broken before 2.9.10
* freeform FQCN actions broken before 2.9.17
* → our collection never worked on Ansible before 2.9.17

Note:
* `ansible.builtin.include_tasks`
* Yes, we still support Python 2.7
* Yes, also 3.12

---

# Ansible

---

## New Galaxy

* Based on `galaxy_ng` codebase that also powers Automation Hub
* The rollout mostly did not affect us, besides
  * deploying new API tokens for our publishing workflows
  * new links (https://galaxy.ansible.com/ui/repo/published/theforeman/foreman/)

---

## Automatic documentation publishing

* We had this last year already
* Now Galaxy also shows collection docs, thanks to the new codebase!

---

# Outlook

---

## CDN configuration

* Yes, this is a carry-over from last year :(
* Katello supports different ways to obtain Red Hat content
* We don't support configuring those
* But that's useful for disconnected setups that use content exports

---

## Getting rid of `requests`

* Ansible has a reasonable HTTP library included
* We could get rid of another external dependency
* Apypie already has support for using alternative implementations
* [PR#1586](https://github.com/theforeman/foreman-ansible-modules/pull/1586)

---

## Argspec for Roles

* Ansible supports role argument specs since 2.11
* Allows validation of passed parameters to roles
* Allows documentation generation
* Is sadly lacking features compared to Module argspec: [ansible#80657](https://github.com/ansible/ansible/issues/80657), [ansible#81734](https://github.com/ansible/ansible/issues/81734), [ansible#81735](https://github.com/ansible/ansible/issues/81735)
* [PR#1602](https://github.com/theforeman/foreman-ansible-modules/pull/1602)

Note:
* #80657 - shared fragments
* #81734 - examples
* #81735 - notes, seealso, requirements, version_added

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)
