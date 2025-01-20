<!--
date: 2021-02-06
link: https://archive.fosdem.org/2021/schedule/event/yourmanagementlayershouldbecattletoo/
-->
# your management layer should be cattle too

Note:
Hi and welcome to my talk "your management layer should be cattle too"

---

## `$ whoami`

Evgeni Golov

Senior Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

:heart: FOSS :heart:

:heart: automation :heart:

Note:
First of all, who am I? I'm a senior software engineer at Red Hat, working on the Foreman project and Red Hat Satellite, especially how to automate workflows around those.

---

> "In the old way of doing things, we treat our servers like pets, for example Bob the mail server. If Bob goes down, it’s all hands on deck. […] In the new way, servers are numbered, like cattle in a herd. For example, www001 to www100. When one server goes down, it’s taken out back, shot, and replaced on the line."
> ([Randy Bias](https://www.slideshare.net/randybias/architectures-for-open-and-scalable-clouds/19), Bill Baker, ~2011)

Note:
* I think everybody heard about the "pet vs cattle" way of handling IT systems over the last 10 years.
* The old way is to care for servers as pets, giving them names, fixing issues as they arise, while the new way is to more agressively replace servers when they start behaving inappropriately.
* But from personal experience, this is not applied to *every* system out there, and I want to change that!

---

## idea

* everything should be repeatable, reproducible and replaceable
    * configuration management
    * immutable infrastructure
* generally applied to servers you have many of ("workers")
* often ignored for systems that exist once (Foreman)

Note:
* The idea is that everything should be easily reproducible and replaceable. We can achieve this with configuration management, immutable infrastructure and others.
* But quite often we only really apply these to systems we have many of - worker systems like webservers.
* The management systems are often ignored, because they exist once and we often don't care to be able to easily replace those.
* As I work on Foreman, my main target here is Foreman, but please, ask more generally "Who configures your configuration management?"

---

## why change?

* You can deploy an identical testing environment
* Or one with minor differences (e.g. other networks)
* Lab environment on your laptop? Sure!
* Rebuild prod from scratch!

Note:
* Why change -- it's an unicorn anyways
* And I would love to have an unicorn pet, really, but not with computers.
* You might want to deploy a testing environment that is sufficiently similar to your prod, or even a second "prod"?
* Or maybe a lab to test out more drastical changes?
* And maybe even a full rebuild of production, because it has grown and shifted so often you're not sure it's safe to run any longer.
* In the hope that I could persuade you to change, let's look *how* to change!

---

## how change?

* Two step process:
    * Step 1: make Foreman installation automated
    * Step 2: make Foreman configuration automated
* Bonus: make all your efforts Open Source so others can benefit!
* We'll use Ansible, but the concepts are applicable everywhere

Note:
* Changing Foreman to be more rebuildable is essentially a two step process:
* First we need to make the initial deployment of the software fully automated
* And then we can continue to focus on the various things you can configure *inside* Foreman to be automated.
* Nice bonus: if you went all that route, you can take your steps, opensource them and make everyone benefit from your expirience -- and that's my job.
* I'll be using Ansible, as I like its simplicity, but you can do the same with other tools too!
* You just won't have the building blocks we prepared for you.

---

# Step 1: make Foreman installation automated

Note:
* So let's go and see what we need to do to automate the deployment

---

## acquire a system to install on

* For lab-on-my-laptop:
    * [Vagrant](https://www.vagrantup.com)
    * [Containers](https://github.com/theforeman/foreman/blob/develop/developer_docs/containers.asciidoc)
* For test/prod:
    * [oVirt/RHV](https://ovirt.org)
    * [Containers](https://github.com/theforeman/foreman/blob/develop/developer_docs/containers.asciidoc)

Note:
* In many environments, Foreman will be responsible for creating VMs, installing them, etc.
* We have to do that by hand once for Foreman itself.
* In the lab case, I'd suggest to use Vagrant, and your-virtualization-of-choice everywhere else.
* The Foreman Project also offers Containers, but I mainly focus on the classical VM here today.

---

## acquire a system to install on

* ideally your lab, your test and your prod use the same technology (container, virt, metal)
* for the demo in this talk we'll use Vagrant (prod: RHV)
* there is currently no container for Katello, so a lot of deployments are classical VMs

Note:
* In my opinion all your environments should use the same technology, to be as bug compatible as possible if you need to try something out.
* For the demo I'll use Vagrant, and our production install is on RHV.
* The main reason to use VMs is the fact that Katello is today not offered as a container.
* Now let's see what's need to install Foreman in the new VM we got

---

## install Foreman

* configure the needed repositories
* install the packages
* execute `foreman-installer`

Note:
* The whole process is rather easy: configure repositories (SCL, Foreman, Plugins), install the packages and rn the installer.
* The installer will then ensure all parts of the system are configured properly to serve Foreman.
* All those steps are not required in a container environment, as the installer isn't used in that case and the container is prepared to just run Foreman.

---

## install Foreman

* enter `theforeman.operations` collection
* goal: easy Foreman operations (installation, upgrade, etc) in VMs
* provided by the Foreman project and *used* by the Foreman project
* "successor" of the content you could find in `theforeman/forklift`, now suited for general consumption

Note:
* We want to make all these steps as easy as possible to our users.
* Therefore we provide an Ansible collection for Foreman Operations, which is supposed to help you with installing and maintaining a Foreman.
* If you tried to manage Foreman with Ansible in the past, you'll notice similarities to the "forklift" repository we have on GitHub: the collection contains roles that evolved from forklift to a more generic use by everyone in the community.
* This evolution allows you to be as close as possible to the best practices the project can think of
* So let's see how a simple installation would look like

---

## install Foreman

```yaml
roles:
  - role: foreman_repositories
    vars:
      foreman_repositories_version: '2.3'
  - role: theforeman.operations.installer
    vars:
      installer_scenario: foreman
```

Note:
* The playbook is rather short. Essentially, two steps: repositories and running the installer
* we just need to tell which version we're deploying and that we're deploying foreman and not some other scenario
* That'd be required if we're about to install Katello…

---

## install Katello

```yaml
roles:
  - role: foreman_repositories
    vars:
      foreman_repositories_version: '2.3'
  - role: katello_repositories
    vars:
      katello_repositories_version: '3.18'
  - role: theforeman.operations.installer
    vars:
      installer_scenario: katello
```

Note:
* We're just adding the right Katello repositories and call the installer with a different scenario.
* Want even more plugins? You sure do!


---

## install more Plugins

```yaml
roles:
  …
  - role: theforeman.operations.installer
    vars:
      installer_scenario: katello
      installer_options:
      - '--enable-foreman-plugin-ansible'
      - '--enable-foreman-proxy-plugin-ansible'
      - '--enable-foreman-plugin-remote-execution'
      - '--enable-foreman-proxy-plugin-remote-execution-ssh'
```

Note:
* Just add the relevant installer flags and it'll do it's magic for us.
* Obviously you can use the same aproach to configure other parts of the installation, not just only plugins.

---

## install Foreman

* at this point we have a Foreman (with plugins) running
* and can continue with adding things *inside* Foreman

Note:
* And that's really it. At this point we have a Foreman up and running and can go on with configuring things *inside* it.

---

# Step 2: make Foreman configuration automated

---

## structured data is key

* if we could describe everything inside Foreman in a structured way, we'd be done
* we can manage a lot with Ansible using the `theforeman.foreman` collection
* modules for managing individual entities inside Foreman
* roles to encapsulate workflows

Note:
* When managing entities in Foreman, it's crucial to be able to store them in a structured way.
* As Foreman doesn't offer this natively, we have Ansible modules that can be fed YAML and manage things for us.
* Those modules live in the `theforeman.foreman` collection on Galaxy
* Together with roles that are trying to encapsulate full workflows (in contrast to single tasks that the modules offer)
* The roles also come with an opiniated data structure, so you don't have to invent your own
* Let's look at some examples

---

## structured data is key

```yaml
- name: create domains
  theforeman.foreman.domain:
    name: "{{ item }}"
  loop:
    - example.com
    - example.org
```

Note:
* This is a trivial example to create two domains inside Foreman
* It's nice as an one-off playbook, but if we need to manage many different entities, this soon will become messy.

---

## structured data is key

`vars.yml`:
```yaml
domains:
  - example.com
  - example.org
```

playbook:
```yaml
- name: create domains
  theforeman.foreman.domain:
    name: "{{ item }}"
  loop: "{{ domains }}"
```

Note:
* If we separate the data and the tasks into different files, things become more manageable again.
* This might sound like too much work for two simple strings, but let's look at the following repositories example

---

## structured data is key

`vars.yml`:
```yaml
products:
- name: CentOS 7
  repositories:
    - name: CentOS 7 Base x86_64
      url: http://mirror.centos.org/centos/7/os/x86_64/
    - name: CentOS 7 Extras x86_64
      url: http://mirror.centos.org/centos/7/extras/x86_64/
    - name: CentOS 7 Updates x86_64
      url: http://mirror.centos.org/centos/7/updates/x86_64/
- name: Foreman Client
  repositories:
    - name: Foreman Client CentOS 7
      url: https://yum.theforeman.org/client/2.3/el7/x86_64/
```

Note:
* repositories in Katello are grouped by Product, so to create a repository, you will have to have at least one Product before
* in this example we create two products: CentOS7 with the Base, Extras and Updates repositories in it, and Foreman Client with the client bits we offer for additional functionality of some plugins
* Accidentally this is also exactly the data structure our role expects you to use, instead of writing an own playbook

---

## structured data is key

playbook:
```yaml
vars_files:
  - vars.yml
roles:
  - role: theforeman.foreman.repositories
```

Note:
* Having the products/repositories defined, the call to actually create them becomes very simple.
* Just call the role and it will do all the steps for you:
    * creating the products first
    * and then adding the repositories to the products

---

## data for a "content consumer"

* products/repositories (`t.f.repositories`)
* content views ([no role yet](https://github.com/theforeman/foreman-ansible-modules/issues/1111))
* lifecycle environments ([role in progress](https://github.com/theforeman/foreman-ansible-modules/pull/1113))
* activation keys (`t.f.activation_keys`)

Note:
* having products and repositories is the very minimum you need for a consumer to use content with katello
* as soon as you want to more granually manage what content a consumer will be able to access, you end up creating lifecycle environments, content views and activation keys
* we offer modules, and in some case also alredy roles that allow you to manage these too

---

## actions for a "content consumer"

* repositories need to be synced
* content views need to be published (if used)
* modules to do this exist, but the *when* greatly varies based on environment

Note:
* just defining "there is content over there" is not sufficient to provide it to clients
* you at least need to synchronize the repositories so that Katello knows which packages it can offer
* and if you use content views, those need regular publishing as they are essentially point in time snapshots of content
* you can (and should) automate those actions, but we need to keep in mind that we're tracking external data, executing the same action at a different time in a different environment might result in slightly differnt content. that's not bad, but one should be aware of that.
* after syncronizing content, we can let a client consume it -- as you will be able to see in the demo later

---

# Step 3: maintenance

Note:
* But wait, there was no step 3 on the agenda?!
* Queue "There always was" meme, because maintenance is always required, just often forgotten to mention and plan for
* And of course we want to provide easy steps for that

---

## upgrading Foreman

* Foreman in a VM means upgrades at some point
* Switch repositories, update packages, run installer

Note:
* If we run Foreman in a VM, at some point we'll have to upgrade it -- roughly every three months.
* The process itself is simple: repositories change, packages update, installer run with DB migrations etc.
* We can just re-use the installation steps with an additional "upgrade packages" in-bewteen

---

## cleaning Katello

* when you use Content Views, old (unused) versions of them accumulate

```yaml
- role: theforeman.foreman.content_view_version_cleanup
  vars:
    content_view_version_cleanup_keep: 10
```

Note:
* When you use content views in Katello, and publish them regularly, you will end up with many content view versions that are unused, but still present in the database
* Ain't nobody needs an old content snapshot from 2 years ago, so we provide a role to help you do some cleaning
* As a bonus: this will drastically improve your Pulp2 to Pulp3 migration, if you already have a Katello installation in house

---

# TBD

operations:
* finalize repository configuration
* proxy deployment (exists in forklift, needs porting/cleaning)

configuration:
* no feature parity with UI/CLI yet
    * especially for provisioning cases that differ per compute

---

# DEMO

---

# Links

* [destructivebuilds repo for the demo](https://github.com/evgeni/destructivebuilds/)
* [forklift](https://github.com/theforeman/forklift/)
* [Foreman Operations Collection](https://github.com/theforeman/foreman-operations-collection/)
* [Foreman Ansible Collection](https://github.com/theforeman/foreman-ansible-modules/)

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-twitter" aria-hidden="true"></i> [@zhenech](https://twitter.com/zhenech)

<i class="fa fa-comment" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)
