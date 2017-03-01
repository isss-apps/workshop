# cicd-pipeline
Demo to introduce CI/CD pipelines on OpenShift.

### Prerequisites

To be able to run this demo locally, please follow the [Minishift documentation](https://github.com/minishift/minishift#installation).


### Usage

Create necessary resources on OpenShift instance.

```
oc create -f fis-image fis-image-streams.json

oc process bc-pipeline.yml | oc create -f -
```
