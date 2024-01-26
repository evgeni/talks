# Generating Ansible modules for REST APIs without AI

---

## `$ whoami`

Matthias Dellweg

Senior Software Engineer at Red Hat

`pulp.squeezer` - 39 modules, one API

---

## `$ whoami`

Evgeni Golov

Principal Software Engineer at Red Hat

`theforeman.foreman` - 82 modules, one API

---

# Motivation

* Writing Ansible modules is a tedious job
* It's a *repetitive* job when you talk to an API
* Computers are great at repeating!

---

## ChatGPT?

* We actually tried!
* It gets the general idea (`requests`, CRUD operations, credentials)
* Even some API interactions were working
* But we couldn't convince it to use any of the existing abstractions (`apypie`, `theforeman.foreman`)

Note:
* We're no Prompt Engineers, we're YAML Engineers after all!

---

## Ansible Lightspeed?

* Can only generate playbooks, not modules

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-envelope" aria-hidden="true"></i> [mdellweg@redhat.com](mailto:mdellweg@redhat.com)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@x9c4@fosstodon.org](https://fosstodon.org/@x9c4)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)

<i class="fa fa-github" aria-hidden="true"></i> [@mdellweg](https://github.com/mdellweg)
