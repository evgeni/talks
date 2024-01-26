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

## Templates (Jinja2)

* Great to provide the boilerplate.
* Needs manual work for the details anyway.
* Reapplying after modifications is complicated.

---

## Code transformations

Once:
`sed -e "s/iteritems/items/g" *.py`

As a compiler:
```
sed -i '/ansible.module_utils.foreman_helper/ s/ansible.module_utils/ansible_collections.theforeman.foreman.plugins.module_utils/g' $(COLLECTION_TMP)/plugins/modules/*.py
sed -i -e '/extends_documentation_fragment/{:1 n; s/- foreman/- theforeman.foreman.foreman/; t1}' $(COLLECTION_TMP)/plugins/modules/*.py
```

---

## Conclusion

When you attempt to consolidate repetitive boilerplate, you create a framework.
`module_utils` and `doc_fragments` are made to create a framework.

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-envelope" aria-hidden="true"></i> [x9c4@redhat.com](mailto:x9c4@redhat.com)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@x9c4@fosstodon.org](https://fosstodon.org/@x9c4)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)

<i class="fa fa-github" aria-hidden="true"></i> [@mdellweg](https://github.com/mdellweg)
