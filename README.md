# cicd-pipeline
Demo to introduce CI/CD pipelines on OpenShift.

### Prerequisites

To be able to run this demo locally, please follow the [Minishift documentation](https://github.com/minishift/minishift#installation).

```
minishift start --iso-url=https://github.com/minishift/minishift-centos-iso/releases/download/v1.0.0-rc.2/minishift-centos7.iso --cpus 2 --memory 4096
```


### Usage

Create necessary resources on OpenShift instance.

```
oc create -f fis-image fis-image-streams.json -n openshift

oc new-project pipeline

oc new-app --template=openshift/jenkins-ephemeral

oc status

oc process bc-pipeline.yml | oc create -f -
```
