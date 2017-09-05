# What is Locker?

Locker is a simple web application for claiming and releasing locks.
It contains a Concourse resource to help make locking various tasks
and jobs in Concourse pipelines easier to manage.

This project is similar to the [pool-resource](https://github.com/concourse/pool-resource), with a few key differences:

1) It uses a webserver with file-backed persistence for the locks. This isn't
   terribly scalable, but since it's handling locks for Concourse pipelines,
   it isn't anticipated to have an enormous traffic load.
2) Pools + lock-names do not need to be pre-defined in order to be used


## How do I use it?

1) Deploy `locker` along side your Concourse database node via the [locker-boshrelease](https://github.com/cloudfoundry-community/locker-boshrelease)
2) Use the `locker-resource` in your Concourse pipelines (see below for resource
   configuration details).

## Deploying locker

To use this bosh release, deploy the manifest:

```
export BOSH_ENVIRONMENT=<alias>
export BOSH_DEPLOYMENT=locker

git clone https://github.com/cloudfoundry-community/locker-boshrelease.git
cd locker-boshrelease

# pick a static IP which will be included in the TLS certificates
locker_host=PICK.AN.IP.ADDRESS

bosh deploy manifests/locker.yml \
  -v "locker-static-ip=$locker_host" \
  --vars-store=tmp/creds.yml
```

If your BOSH does have Credhub/Config Server, then you do not require the  ` --vars-store` flag. Credhub will manage the generation of passwords and certificates at deploy time. Use the `credhub` CLI to get the credentials.

This deployment will provision a dedicated VM running locker.

To interact with your Locker API:

```
locker_password=$(bosh int tmp/creds.yml --path /locker-password)
locker_ca="$(bosh int tmp/creds.yml --path /locker-tls/ca)"
curl --cacert <(echo "$locker_ca") -u locker:$locker_password https://${locker_host:?required}:8910/locks
```

Initially this will return `{}`. As you claim locks, it will return the status of all locks.

### Concourse integration

The original use case for `locker` was to be integrated with [Concourse](https://concourse.ci).

For this scenario, this `locker` job should probably be colocated on the database node of your concourse deployment. Ensure that the node that `locker` is being installed on has both a static IP, and persistent disk. The disk is used to persist lock data across restarts/stemcell upgrades. The static IP is required to provide the `locker-resource` in Concourse with a reliable URL to contact. Lastly, ensure that the username, password, and SSL cert/key attributes are all set when running in production.

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

### Use Cloud Foundry routing

Instead of requiring a static IP, connecting to a non-443 port, and managing internal TLS certificates you can opt to register `locker` with the Cloud Foundry Routing mesh.


```
cf_deployment=cf
system_domain=$(bosh -d $cf_deployment manifest | bosh int - --path /instance_groups/name=api/jobs/name=cloud_controller_ng/properties/system_domain)

bosh deploy manifests/locker.yml \
   --vars-store tmp/creds.yml \
  -o manifests/operators/routing.yml \
  -v routing-nats-deployment=$cf_deployment \
  -v "locker-uri=locker.$system_domain"

locker_password=$(bosh int tmp/creds.yml --path /locker-password)
curl -u locker:$locker_password https://locker.$system_domain/locks
```
