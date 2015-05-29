# BOSH OpenStack CPI release

This is a [BOSH](http://bosh.io/) release for the external [BOSH OpenStack CPI](https://github.com/frodenas/bosh-openstack-cpi/).

## Disclaimer

This is **NOT** presently a production ready [BOSH OpenStack CPI](https://github.com/frodenas/bosh-openstack-cpi/) BOSH release. This is a work in progress. It is suitable for experimentation and may not become supported in the future.

This is **NOT** an official BOSH OpenStack CPI BOSH release, and it is NOT intended to be compatible with the official one.

The official and supported BOSH OpenStack CPI BOSH release can be found at [http://bosh.io/releases/github.com/cloudfoundry-incubator/bosh-openstack-cpi-release](http://bosh.io/releases/github.com/cloudfoundry-incubator/bosh-openstack-cpi-release).

## Usage

I am assuming you are familiar with [BOSH](http://bosh.io/) and its terminology. If not, please take a look at the [BOSH documentation](http://bosh.io/docs) before running this procedure.

### Setup the [OpenStack](https://www.openstack.org/) environment

See [Prepare an OpenStack environment](http://bosh.io/docs/init-openstack.html#prepare-openstack) instructions.

### Install the bosh-init CLI

Install the [bosh-init](https://bosh.io/docs/install-bosh-init.html) tool in your workstation.

### Create a deployment directory

Create a deployment directory to store all `bosh-init` artifacts:

```
$ mkdir openstack-bosh-deployment
$ cd openstack-bosh-deployment
```

### Create a deployment manifest

Create a `openstack-bosh-manifest.yml` deployment manifest file inside the previously created deployment directory with the following properties:

```
---
name: bosh

releases:
  - name: bosh
    url: https://bosh.io/d/github.com/cloudfoundry/bosh?v=168
    sha1: 57320a93a7a15e51af4c57d9d9c22706c45c9953
  - name: bosh-openstack-cpi
    url: http://storage.googleapis.com/bosh-stemcells/bosh-openstack-cpi-1.tgz
    sha1: 31f56d358e36d8665b1d77cd5eea764e219af576

resource_pools:
  - name: vms
    network: private
    stemcell:
      url: https://bosh.io/d/stemcells/bosh-openstack-kvm-ubuntu-trusty-go_agent?v=2950
      sha1: 7cd2a246ad24b2e8116686a89c5bd97fff815763
    cloud_properties:
      flavor: m1.xlarge

disk_pools:
  - name: disks
    disk_size: 32_768
    cloud_properties:
      type: standard

networks:
  - name: private
    type: manual
    subnets:
      - range: __PRIVATE_CIDR__ # <--- Replace with a private subnet CIDR
        gateway: __PRIVATE_GATEWAY_IP__ # <--- Replace with a private subnet's gateway
        cloud_properties:
          network: __OS_NETWORK_NAME__ # <--- # Replace with your OpenStack Network name

  - name: public
    type: vip

jobs:
  - name: bosh
    instances: 1

    templates:
      - name: nats
        release: bosh
      - name: redis
        release: bosh
      - name: postgres
        release: bosh
      - name: powerdns
        release: bosh
      - name: blobstore
        release: bosh
      - name: director
        release: bosh
      - name: health_monitor
        release: bosh
      - name: cpi
        release: bosh-openstack-cpi
      - name: registry
        release: bosh-openstack-cpi

    resource_pool: vms
    persistent_disk_pool: disks

    networks:
      - name: private
        static_ips:
         - __PRIVATE_IP__ # <--- Replace with the private IP
        default:
          - dns
          - gateway
      - name: public
        static_ips:
          - __FLOATING_IP__ # <--- Replace with the floating IP

    properties:
      nats:
        address: 127.0.0.1
        user: nats
        password: nats

      redis:
        listen_address: 127.0.0.1
        address: 127.0.0.1
        password: redis

      postgres: &db
        adapter: postgres
        host: 127.0.0.1
        user: postgres
        password: postgres
        database: bosh

      dns:
        address: __PRIVATE_IP__ # <--- Replace with the private IP
        domain_name: microbosh
        db: *db
        recursor: 8.8.8.8

      blobstore:
        address: __PRIVATE_IP__ # <--- Replace with the private IP
        provider: dav
        director:
          user: director
          password: director
        agent:
          user: agent
          password: agent

      director:
        address: 127.0.0.1
        name: micro-openstack
        db: *db
        cpi_job: cpi
        max_threads: 3

      hm:
        http:
          user: hm
          password: hm
        director_account:
          user: admin
          password: admin
        resurrector_enabled: true

      ntp: &ntp
        - 0.pool.ntp.org
        - 1.pool.ntp.org

      openstack: &openstack_properties
        identity_endpoint: __GCE_PROJECT__ # <--- Replace with your OpenStack Identify endpoint URI
        username: __OS_USERNAME__ # <--- Replace with your OpenStack Username
        user_id: __OS_USERID__ # <--- Replace with your OpenStack User ID
        password: __OS_PASSWORD__ # <--- Replace with your OpenStack Password
        api_key: __OS_API_KEY__ # <--- Replace with your OpenStack API Key
        tenant_name: __OS_TENANT_NAME__ # <--- Replace with your OpenStack Tenant Name
        tenant_id: __OS_TENANT_ID__ # <--- Replace with your OpenStack Tenant ID
        domain_name: __OS_DOMAIN_NAME__ # <--- Replace with your OpenStack Domain Name
        domain_id: __OS_DOMAIN_ID__ # <--- Replace with your OpenStack Domain ID
        region: __OS_REGION__ # <--- Replace with your OpenStack Username
        default_keypair: __OS_DEFAULT_KEYPAI__ # <--- Replace with your OpenStack Username
        default_security_groups:
          - bosh # <--- Replace with your default OpenStack Security Groups to be used when creating servers
        disable_config_drive: false # <--- Replace if you want to disable injecting OpenStack user data via the Config Drive
        disable_neutron: false # <--- Replace if you want to disable OpenStack Neutron interactions
        ignore_server_availability_zone: false # <--- Replace if you want to ignore OpenStack Server's Availability Zone when creating OpenStack volumes

      agent:
        mbus: nats://nats:nats@__PRIVATE_IP__:4222 # <--- Replace with the private IP
        ntp: *ntp
        blobstore:
           options:
             endpoint: http://__PRIVATE_IP__:25250 # <--- Replace with the private IP
             user: agent
             password: agent

      registry:
        host: __PRIVATE_IP__ # <--- Replace with the private IP
        username: registry
        password: registry

cloud_provider:
  template:
    name: cpi
    release: bosh-openstack-cpi

  ssh_tunnel:
    host: __FLOATING_IP__ # <--- Replace with the floating IP
    port: 22
    user: vcap
    private_key: __PRIVATE_KEY_PATH__ # <--- Replace with the location of your BOSH SSH private key

  mbus: https://mbus:mbus@__FLOATING_IP__:6868 # <--- Replace with the floating IP

  properties:
    openstack: *openstack_properties
    agent:
      mbus: https://mbus:mbus@0.0.0.0:6868
      ntp: *ntp
      blobstore:
        provider: local
        options:
          blobstore_path: /var/vcap/micro_bosh/data/cache

```

### Deploy

Using the previously created deployment manifest, now we can deploy it:

```
$ bosh-init deploy openstack-bosh-manifest.yml
```

## Contributing

In the spirit of [free software](http://www.fsf.org/licensing/essays/free-sw.html), **everyone** is encouraged to help improve this project.

Here are some ways *you* can contribute:

* by using alpha, beta, and prerelease versions
* by reporting bugs
* by suggesting new features
* by writing or editing documentation
* by writing specifications
* by writing code (**no patch is too small**: fix typos, add comments, clean up inconsistent whitespace)
* by refactoring code
* by closing [issues](https://github.com/frodenas/bosh-openstack-cpi-boshrelease/issues)
* by reviewing patches

### Submitting an Issue
We use the [GitHub issue tracker](https://github.com/frodenas/bosh-openstack-cpi-boshrelease/issues) to track bugs and features.
Before submitting a bug report or feature request, check to make sure it hasn't already been submitted. You can indicate
support for an existing issue by voting it up. When submitting a bug report, please include a
[Gist](http://gist.github.com/) that includes a stack trace and any details that may be necessary to reproduce the bug,
including your gem version, Ruby version, and operating system. Ideally, a bug report should include a pull request with
 failing specs.

### Submitting a Pull Request

1. Fork the project.
2. Create a topic branch.
3. Implement your feature or bug fix.
4. Commit and push your changes.
5. Submit a pull request.

### Create new release

#### Creating a final release

If you need to create a new final release, you will need to get read/write API credentials to the [@cloudfoundry-community](https://github.com/cloudfoundry-community) s3 account.

Please email [Dr Nic Williams](mailto:&#x64;&#x72;&#x6E;&#x69;&#x63;&#x77;&#x69;&#x6C;&#x6C;&#x69;&#x61;&#x6D;&#x73;&#x40;&#x67;&#x6D;&#x61;&#x69;&#x6C;&#x2E;&#x63;&#x6F;&#x6D;) and he will create unique API credentials for you.

Create a `config/private.yml` file with the following contents:

``` yaml
---
blobstore:
  s3:
    access_key_id:     ACCESS
    secret_access_key: PRIVATE
```

You can now create final releases for everyone to enjoy!

```
bosh create release
# test this dev release
git commit -m "updated BOSH OpenStack CPI release"
bosh create release --final
git commit -m "creating vXYZ release"
git tag vXYZ
git push origin master --tags
```

## Copyright

See [LICENSE](https://github.com/frodenas/bosh-openstack-cpi-boshrelease/blob/master/LICENSE) for details.
Copyright (c) 2015 Ferran Rodenas.
