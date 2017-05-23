# What is Locker?

Locker is a simple web application for claiming and releasing locks.
It contains a Concourse resource to help make locking various tasks
and jobs in Concourse pipelines easier to manage.

This project is similar to the [pool-resource](https://github.com/concourse/pool-resource), with a few key differences:

1) It uses a webserver with file-backed persistence for the locks. This isn't
   terribly scalable, but since it's handling locks for Concourse pipelines,
   it isn't anticipated to have an enormous traffic load.
2) Pools + lock-names do not need to be pre-defined in order to be used

# How do I use it?

1) Deploy `locker` along side your Concourse database node via the [locker-boshrelease](https://github.com/cloudfoundry-community/locker-boshrelease)
2) Use the `locker-resource` in your Concourse pipelines (see below for resource
   configuration details).

# BOSH Release for locker

To use this bosh release, first upload it to your bosh:

```
bosh target BOSH_HOST
git clone https://github.com/cloudfoundry-community/locker-boshrelease.git
cd locker-boshrelease
bosh upload release releases/locker/locker-1.yml
```

For [bosh-lite](https://github.com/cloudfoundry/bosh-lite), you can quickly create a deployment manifest & deploy a cluster. Note that this requires that you have installed [spruce](https://github.com/geofffranks/spruce).

```
templates/make_manifest warden
bosh -n deploy
```

The templates located in `templates/` are there for posterity, to deploy manually to a bosh-lite for testing.
They also require cloud config (warden sample is in `templates/`).

However, in practice, this job should probably be colocated on the database node of your concourse
deployment. Ensure that the node that `locker` is being installed on has both a static IP, and persistent
disk. THe disk is used to persist lock data across restarts/stemcell upgrades. The static IP is required
to provide the `locker-resource` in Concourse with a reliable URL to contact. Lastly, ensure that the username,
password, and SSL cert/key attributes are all set when running in production.

```
instance_groups:
- name: db
  instances: 1
  stemcell: ubuntu
  persistent_disk_pool: my-persistent-disk-pool #locker requires persistent disk
  networks:
  - name: concourse
    static_ips: [ 10.10.10.10 ] # locker requires a static IP so the locker-resource knows what to talk to
  jobs:
  - name: postgresql
    release: concourse
    properties:
     databases:
     - name: atc
       role: atc
       password: your-random-password-here
  - name: locker
    release: locker
    properties:
      locker:
        user: locker_username
        password: your-other-random-password-here
        ssl_cert: |
          SSL_CERT_HERE
        ssl_key: |
          SSL_KEY_HERE
```
