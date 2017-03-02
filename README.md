# workshop
Workshop instructions for System Integration lecture

### Prerequisites

Both binaries should be placed on `$PATH` depending on your OS.

#### Minishift
[Download latest beta](https://github.com/minishift/minishift/releases/tag/v1.0.0-beta.4)

[Installation instructions](https://github.com/minishift/minishift#installation)

```
minishift start --iso-url=https://github.com/minishift/minishift-centos-iso/releases/download/v1.0.0-rc.2/minishift-centos7.iso --cpus 2 --memory 4096
```

#### OC client
[Download latest stable](https://github.com/openshift/origin/releases/tag/v1.4.1)

## cicd-pipeline
Demo to introduce CI/CD pipelines on OpenShift.

### Instructions

Create necessary resources on OpenShift instance.

```
oc create -f fis-image fis-image-streams.json -n openshift

oc new-project pipeline

oc new-app --template=openshift/jenkins-ephemeral

oc status

oc process bc-pipeline.yml | oc create -f -
```


### Webhooks
Builds can be invoked via webhooks (generic or GitHub). See [oficial documentation](https://docs.openshift.com/container-platform/3.4/dev_guide/builds/triggering_builds.html#generic-webhooks) for more details.


Locate webhook URL from ouput of this command.
```
oc describe bc cicd
```

Start build with `POST` method on provided URL.
```
curl -k -X POST <webhook-url>
