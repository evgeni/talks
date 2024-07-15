# From Jenkins to GitHub Actions
## lessons, problems, outlook

---

## `$ whoami`

Evgeni Golov

Principal Software Engineer at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

---

## Foreman?!

~~"Foreman is a complete lifecycle management tool for physical and virtual servers."~~

* ~50 Foreman plugins
* ~25 Foreman Proxy modules/providers
* ~20 Hammer plugins
* LOTS OF TESTS!
* [foreman-plugin-overview](https://theforeman.github.io/foreman-plugin-overview/)

Note:

* I couldn't resist to add this quote, but it's really not what we'll be talking about today.

---

## CI landscape in 2023

* Jenkins with 142 jobs (+ second with ~30)
* "GitHub Pull Request Builder" plugin
* Access to `theforeman` and `Katello` organizations
* Others (`ATIX-AG`, `betadots`, `dm-drogeriemarkt`, …) need own solutions
* Big migrations planned: Ruby 3, Webpack 5

Note:

* Obviously, not 142 PR-related jobs!
* GHPRB unmaintained!
* Centrally managed by the infra-team
* Plugin maintainers have no easy way to influence the testing
* CentOS CI hosts another Jenkins for us, which we ignore today. 34 Jobs.

---

## CI landscape in ~~2023~~ 2015

```
commit 22d2bc028210b516894a20c04c638c942e1f9c7c
Author: Dominic Cleal <dcleal@redhat.com>
Date:   Tue Jun 2 13:14:27 2015 +0100

    Add test_plugin_* template and move bootdisk
```

* WAT?!
* [Allow use of Jenkinsfiles #1683](https://github.com/theforeman/foreman-infra/issues/1683) - 2021
* Several plugins were already using self-made GHA workflows

Note:

* Since 2015 multiple refactorings happened (Shell to Groovy!), but the overall design remained.

---

## CI landscape in 2024

* Jenkins with 90 jobs
* Second one untouched
* GitHub Actions used in 35 (Foreman plugin) repositories
* more tools = better?

Note:

* https://github.com/theforeman/actions/network/dependents

---

## `@theforeman/actions`

* Shared workflows for Foreman-related things
* `foreman_plugin.yml`
* `foreman_plugin_js.yml`
* `rubocop.yml`
* `test-gem.yml`
* `release-gem.yml`

---

## `foreman_plugin.yml`

* Every plugin should use that!
* Automatically detects the right Ruby/NodeJS/PostgreSQL versions
* Executes `test:${plugin_name}`, `db:seed`, `plugin:assets:precompile`
* Additionally executes plugin migrations *after* full Foreman install

Note:

* `matrix.json` in foreman.git
* Runs PostgreSQL in a container
* Installs Ruby/NodeJS and gems/packages
* Tasks can be configured
* Generates many jobs, we have high limits due to sponsoring

---

## `matrix.json`

```json
{
  "postgresql": ["13"],
  "ruby": ["2.7", "3.0"],
  "node": ["14", "18"]
}
```

* Consumed by `theforeman/gha-matrix-builder`
* Makes it easy to adjust many consumers in one go

---

## `foreman_plugin.yml`

```yaml
  test:
    name: Ruby
    uses: theforeman/actions/.github/workflows/
          foreman_plugin.yml@v0
    with:
      plugin: MY_PLUGIN
```

Note:

* Sorry for the line-break
* `theforeman/gha-matrix-builder` called internally, you don't have to care

---

## `foreman_plugin.yml`

Testing against multiple Foreman versions

```yaml
  test:
    strategy:
      matrix:
        foreman:
          - 3.9-stable
          - develop
    uses: theforeman/actions/.github/workflows/
          foreman_plugin.yml@v0
    with:
      plugin: MY_PLUGIN
      foreman_version: ${{ matrix.foreman }}
```

Or configuring your stable branches to use a stable Foreman branch!

---

## `foreman_plugin.yml`

Testing a plugin against a PR for Foreman

(Ruby or Rails upgrades)

```yaml
  test:
    name: Ruby
    uses: theforeman/actions/.github/workflows/
          foreman_plugin.yml@v0
    with:
      plugin: MY_PLUGIN
      foreman_version: refs/pull/1234/head
```

Note:

* Used that for the Ruby 3.0 update
* Using it for the Zeitwerk work for Rails 7
* You can do that yourself!

---

## CI the CI

This was almost impossible in Jenkins

```yaml
on:
  pull_request:
    paths:
      - '.github/workflows/foreman_plugin.yml'
```

```yaml
jobs:
  test:
    name: Ruby
    uses: ./.github/workflows/foreman_plugin.yml
    with:
      plugin: foreman-tasks
      plugin_repository: theforeman/foreman-tasks
```

🤯

---

## `foreman_plugin_js.yml`

* Every plugin with JS tests should use that!
* Automatically detects the right Ruby/NodeJS versions
* Executes `npm lint`, `npm test`
* Can do all the nice tricks `foreman_plugin.yml` can

Note:

* Yes, Ruby is required, but luckilly not all the gems

---

## `test-gem.yml`

* Can be used by any Ruby gem
* So Hammer and Foreman Proxy plugins
* Detects Ruby versions from the gemspec

Note:

* Not yet widely used, compared to `foreman_plugin.yml`
* Some discussion ongoing whether we want a `matrix.json`

---

## `rubocop.yml`

* Runs Rubocop
* Often used as a pre-requisite for main tests

---

## `gem-release.yml`

* Builds the gem
* Pushes it to Rubygems.org
* Needs a secret to be defined in your repository settings

---

## CI landscape in 2024
## the bad things

* No more `[test katello]` comments
* Only people with write permission can re-trigger tests
* Test result reporting is different to Jenkins
  * GH supports annotations
  * Shown in the file tab of a PR
  * Not all steps report that correctly yet

Note:

* GH view can be better to link failures to code
* This wasn't possible with Jenkins, so not everything is bad

---

## CI landscape in 2025

* Get rid of second Jenkins
* Standardize running select core tests for Plugins
* Standardize Foreman Proxy plugin tests
* Standardize Hammer plugin tests

Note:

* Second Jenkins was needed to access CentOS CI API, it's openly available now
* The indirection costs time and makes test results less readable

---

## CI landscape in 2025

* Get rid of main Jenkins?
  * GH is bad for storing artifacts
  * GH is bad for time-triggered jobs
  * = We'd like to, but probably not

Note:

* artifacts needed for nightly
* time-triggers are needed for release pipelines, esp nightly

---

## CI landscape in 2025

* `@theforeman/actions` for GitLab?
  * ATIX uses GitLab internally
  * Red Hat uses GitLab internally

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)

---

# Links

* [foreman-plugin-overview](https://theforeman.github.io/foreman-plugin-overview/)
* [`@theforeman/actions`](https://github.com/theforeman/actions)
* [`theforeman/gha-matrix-builder`](https://github.com/theforeman/gha-matrix-builder)
