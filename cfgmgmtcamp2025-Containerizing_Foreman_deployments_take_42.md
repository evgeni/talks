<!--
date: 2025-02-04
link: https://cfp.cfgmgmtcamp.org/ghent2025/talk/WLFEVJ/
-->
# Containerizing Foreman deployments
## take #42

<!--

I was asked to submit a Steve Ballmer style "Automation! Automation! Automation!" lightning talk, but that's really not my style.

So let's instead talk about containers!

Especially containers for Foreman.
Suiteable for running in production, with plugins and auxilary services like Candlepin and Pulp.
Running like normal system services with Podman and systemd or on your Kubernetes cluster.

We've had a `Dockerfile` in the main Foreman repository for over 5 years (May 2019), have been publishing it to Quay for a long time and I've heard people actually been using it. But it's not flexible (no plugins!), mainly aimed at developers and not well maintained overall (no CI until 2023!).

In this talk we will present the current iteration (luckily not actually #42!) of a possible design for running a production Foreman with plugins, bells and whistles in a container environment. We will also discuss what this (probably) means for future deployments on Foreman and upgrades of existing setups.

-->

---

## `$ whoami`

Evgeni Golov

Principal Software Engineer at Red Hat

ex-Consultant at Red Hat

Debian Developer

♥ FOSS ♥

♥ automation ♥

Note:
With that sorted, let's see why we all gathered hered today

---

* 2009: Foreman gets created
* 2013: Docker gets created
* 2019: Ohad adds a `Dockerfile` to `foreman.git`
* 2025: It's still not possible to run Foreman as a container in production

Note:
* Yes, we had containers before Docker
* No, we didn't ship software as containers before that
* So, if it's not possible, what do people do instead?

---

## classical Foreman deployment

* RPM and DEB packages
* orchestrated by Installer/Puppet
* users have lots of control (OS, plugins, etc)

Note:
* RPM packages also serve as a base for Red Hat Satellite and ATIX Orcharino products
* If you happen to be a Debian developer, don't look too closely at our Debian packages ;-)
* But I said there is a Dockerfile, what's wrong with that?

---

## existing `Dockerfile`

* built from source
* no orchestration
* no plugins
* no control

Note:
* We build containers for stable branches
* We don't test the result
* Heck, we didn't even test that it builds until 2023
* The original aim was to spin up a Foreman for development
* We have no data how many people run this in "production", but can't be too many

---

## everyone wants Kubernetes

<span class="fragment">or at least Podman</span>

---

## 2024: the research

* <!-- {_class="fragment"} --> let's throw <em>everything</em> away and start fresh
* <!-- {_class="fragment"} --> maybe keep (RPM) packages?
* <!-- {_class="fragment"} --> make it work with Podman, Kubernetes later

Note:
* If you never looked inside our Debian packages: don't!
* We need to know what we ship (SBOM anyone?), RPM is already well tested
* Obviously, you can run the containers with whatever OCI runtime you want
* We focus on Podman first

---

## Why podman?

* We still need "one VM all services" deployment
* Podman (since 5.0) has very good systemd integration
* Can use Kubernetes YAML as input

Note:
* We need to accomodate for existing setups and admins
* If we keep the "feel" similar, they will have an easier transition
* Kube YAML would allow us to keep the same container definitions in the future
* Podman 5.0 is in CentOS Stream 9, Debian Trixie 13 and Ubuntu 24.10+

---

## Which services?

![foreman architecture](https://theforeman.org/static/images/foreman_architecture.png)

Note:
* We all have seen this picture in the docs
* Today, we only care about the yellow part in the middle

---

## Which services?

<div class="mermaid">
  <pre>
classDiagram
    Foreman ..> Foreman_Proxy
    Foreman ..> Candlepin
    Foreman ..> Pulp
    Foreman ..> Redis
    Pulp ..> Redis
    httpd ..> Foreman
    httpd ..> Pulp
    Pulp ..> PostgreSQL
    Foreman ..> PostgreSQL
    Candlepin ..> PostgreSQL
    dynflow_orchestrator ..> PostgreSQL
    dynflow_orchestrator ..> Redis
    dynflow_worker ..> PostgreSQL
    dynflow_worker ..> Redis
    dynflow_worker ..> Pulp
    dynflow_worker ..> Candlepin
  </pre>
</div>

Note:
* Yes, this is more than "core"
* We need dynflow for a working tasks system
* We need Pulp/Candlepin for Katello
* Each of those is a systemd unit today
* Technically, we can replace them one by one and nobody will notice?
* FIXME: Pulp has sub-services!

---

## Candlepin container

```[|1|2|3]
FROM quay.io/centos/centos:stream9
RUN dnf -y install candlepin
CMD ["/usr/libexec/tomcat/server", "start"]
```

Note:
* Candlepin is a very isolated service on its own
* So it's perfect as the first stab at re-deployment
* I skipped the repo setup above for brevity
* There are no "moving" parts, no plugins, no modifications

---

## Candlepin container

```[|5|6|7-9|11-13]
# cat /etc/containers/systemd/candlepin.container
[Container]
ContainerName=candlepin
HostName=quadlet.example.com
Image=quay.io/ehelms/candlepin:4.4.14
Network=host
Secret=candlepin-ca-cert,
       target=/etc/candlepin/certs/candlepin-ca.crt,
       mode=0440,type=mount
…
Secret=candlepin-candlepin-conf,
       target=/etc/candlepin/candlepin.conf,
       mode=0440,type=mount
…
```

Note:
* We use `Network=Host` so that services can interact like they were running without a container
* Candlepin takes care of database migrations on its own
* Configuration and certificates are mounted as secrets
* Generation and management of these happens using Ansible
* We aren't sure it will remain Ansible
* Ideally we'd not have to generate the whole config file

---

## Candlepin container

```console
# systemctl status candlepin
● candlepin.service
     Loaded: loaded
             (/etc/containers/systemd/candlepin.container;
             generated)
     Active: active (running)
   Main PID: 1330 (conmon)
     Memory: 975.1M
        CPU: 19.567s
     CGroup: /system.slice/candlepin.service
             ├─libpod-payload-ef42219c81f60b6287df9caff
             │ └─1346 /usr/lib/jvm/jre-17/bin/java …
             └─runtime
               └─1330 /usr/bin/conmon …
```

Note:
* behaves like a normal systemd unit
* logs stdout/stderr to journal

---

## Foreman container

```[|3-4|6-10|12-14]
FROM quay.io/centos/centos:stream9

RUN dnf install -y foreman foreman-postgresql\
    foreman-service foreman-redis foreman-dynflow-sidekiq

ARG FOREMAN_PLUGINS="foreman-tasks\
                     foreman_remote_execution\
                     katello"

RUN for PLUGIN in ${FOREMAN_PLUGINS}; do … ; done

CMD /usr/share/foreman/bin/rails db:migrate &&\
    /usr/share/foreman/bin/rails db:seed &&\
    /usr/share/foreman/bin/rails server -e production
```

Note:
* Installing `foreman-dynflow-sidekiq` allows to re-use the same container for Dynflow
* If we install all plugins here, they all get automatically enabled, not everyone wants this
* Current idea: install the packages, but only load/enable if some environment variable is set
* If you want other plugins baked in, it's easier to rebuild now
* We need to take care of database migrations (is that the best way?)

---

## Foreman container

```[|7]
# cat /etc/containers/systemd/foreman.container
[Container]
ContainerName=foreman
HostName=quadlet.example.com
Image=quay.io/evgeni/foreman-rpm:nightly
Network=host
Secret=foreman-database-url,type=env,target=DATABASE_URL
Secret=foreman-settings-yaml,type=mount,
       target=/etc/foreman/settings.yaml
…
Secret=foreman-client-cert,type=mount,
       target=/etc/foreman/client_cert.pem
…
```

Note:
* Conceptually, this looks exactly like the Candlepin container
* One noteworthy new thing is `type=env`
* And yes, this is missing passing in the plugins variable

---

## Dynflow containers

```[|1,3|5|8-10]
# cat /etc/containers/systemd/dynflow-sidekiq@.container
[Container]
ContainerName=dynflow-sidekiq-%i
HostName=quadlet.example.com
Image=quay.io/evgeni/foreman-rpm:nightly
Network=host
Secret=…
Exec=/usr/libexec/foreman/sidekiq-selinux -e production \
     -r /usr/share/foreman/extras/dynflow-sidekiq.rb \
     -C /etc/foreman/dynflow/%i.yml
```

Note:
* We can do template containers! Just like systemd
* This can be instantiated as `@orchestrator`, `@worker` etc
* It's actually using the very same container image as the Foreman container
* as it's mainly the same code running, just a different entry point

---

## Pulp containers

* Built by the Pulp Project!
* All-in-One (incl PostgreSQL and Redis)
* API / Content / Worker

Note:
* During the research we first started with AIO
* Switched to dedicated as it better matches our architecture
* I'll save the code snippets, they're boring

---

## Which containers?

<div class="mermaid">
  <pre>
classDiagram
    Foreman ..> Foreman_Proxy
    Foreman ..> Candlepin
    Foreman ..> Pulp
    Foreman ..> Redis
    Pulp ..> Redis
    httpd ..> Foreman
    httpd ..> Pulp
    Pulp ..> PostgreSQL
    Foreman ..> PostgreSQL
    Candlepin ..> PostgreSQL
    dynflow_orchestrator ..> PostgreSQL
    dynflow_orchestrator ..> Redis
    dynflow_worker ..> PostgreSQL
    dynflow_worker ..> Redis
    dynflow_worker ..> Pulp
    dynflow_worker ..> Candlepin

    style Foreman fill:#f96
    style Foreman_Proxy fill:#096
    style Pulp fill:#f96
    style Candlepin fill:#f96
    style dynflow_orchestrator fill:#f96
    style dynflow_worker fill:#f96
  </pre>
</div>

Note:
* Foreman Proxy is only here for Katello/Pulp communication, not a full proxy
* The containers run as systemd services, so start/stop/journal still works

---

## are we there yet?

<span class="fragment">No.</span>
<span class="fragment">But we know which questions we need to answer.</span>

---

## installation / configuration

* our research used Ansible
* our current Installer is Puppet
* is this a chance to simplify?

Note:
* "everything is a container" could make things easier to deploy
* can we configure more things using environment variables?
* but in reality we're not there yet

---

## migrations / upgrades

* existing setups will require migration
* the installer decision will influence this

---

## logging

* Applications log to `/var/log`, but we don't mount that
* stdout/stderr is correctly collected by systemd

Note:
* We need to ensure everything logs to stdout so it can be collected
* Pulp already does that
* Foreman can be easily configured
* Candlepin/Tomcat TBD

---

## PostgreSQL

* Should we move PostgreSQL into a container?
* How will we handle PostgreSQL upgrades?

Note:
* Or maybe say "DB is provided by the environment"?

---

## HTTP ingress

* Today we use Apache httpd from the host
* Inherits crypto policies from the host
* Inherits Kerberos setup from the host

Note:
* On K8s you probably want to use the native Ingress?

---

## packaging

* we're using RPM packages as the base for the containers
* the containers can run on Debian/Ubuntu
* containers *can* be build from source

Note:
* I am not saying we're dropping Debian packaging, but we might, it's not good anyway

---

## integration

* Foreman uses the Foreman Proxy to integrate with services
* We avoided that part in our research
* We'll also probably avoid it for the first real deployment

Note:
* We knew the question would come, but there are too many moving parts

---

## development setup

* "quickly apply this patch" doesn't work anymore
* neither does `bundle exec rails`
* we need to get dev and prod deployments closer together

---

## you

* Pretty sure we didn't find all the questions
* There is an [RFC](https://community.theforeman.org/t/rfc-foreman-production-installation-via-containers-and-podman-quadlets/40611/)
* We're also eager to hear ideas!

---

# Links

* [podman-systemd](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
* [foreman-quadlet](https://github.com/theforeman/foreman-quadlet)
* [RFC: Foreman Production Installation via Containers and Podman Quadlets](https://community.theforeman.org/t/rfc-foreman-production-installation-via-containers-and-podman-quadlets/40611/)

---

# Thanks!

<i class="fa fa-envelope" aria-hidden="true"></i> [evgeni@golov.de](mailto:evgeni@golov.de)

<i class="fa fa-globe" aria-hidden="true"></i> [die-welt.net](https://www.die-welt.net)

<i class="fa fa-mastodon" aria-hidden="true"></i> [@zhenech@chaos.social](https://chaos.social/@zhenech)

<i class="fa fa-github" aria-hidden="true"></i> [@evgeni](https://github.com/evgeni)
