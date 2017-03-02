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

#### Install Fuse Integration Services image streams

```
oc login https://`minishift ip`:8443 -u system:admin
oc create -f fis-image-streams.json -n openshift
```

Login as developer from now on

```
oc login --username=developer --password=developer
```

## Deploying to OpenShift

...

### Deploying PostgreSQL

```
oc process openshift//postgresql-ephemeral POSTGRESQL_USER=student POSTGRESQL_PASSWORD=isss-secret POSTGRESQL_DATABASE=isss | oc create -f -
```

Use oc get pods to see the actual generated pod name

```
$ oc get pods
NAME                 READY     STATUS    RESTARTS   AGE
postgresql-1-ky8z8   1/1       Running   0          12s
```

Use oc rsh to connect to the running PostgreSQL

```
oc rsh postgresql-1-ky8z8
```

Run psql client from inside the pod to execute SQL

```
psql isss
```

Create the database and the mock data

```
CREATE TABLE items (
    id             integer PRIMARY KEY,
    title          varchar(40) NOT NULL,
    stock          integer NOT NULL,
    supplier_days  integer NOT NULL,
    supplier_id    integer
);

CREATE TABLE suppliers (
    supplier_id            integer PRIMARY KEY,
    name                   varchar(40) NOT NULL,
    address                varchar(90) NOT NULL 
);

insert into suppliers (supplier_id, name, address)  values (5000, 'Horsin Around', 'Ponnyville, 456 01 FT');
insert into suppliers (supplier_id, name, address)  values (5001, 'Instruments Big and Small', 'Fluteville, 43 Pan St. Neverland');
insert into suppliers (supplier_id, name, address)  values (5002, 'Aperture Sci.', '42 Science st, Next to Black Mesa, CA');

insert into items (id, title, stock, supplier_days, supplier_id) values (1000, 'BoJack Horseman Plush Toy', 42, 5, 5000);
insert into items (id, title, stock, supplier_days, supplier_id) values (1001, 'Princess Carolyn Plush Toy', 33, 5, 5000);
insert into items (id, title, stock, supplier_days, supplier_id) values (1002, 'Diane Nguyen Plush Toy', 0, 15, 5000);

insert into items (id, title, stock, supplier_days, supplier_id) values (1003, 'Pan Flute', 5, 7, 5001);
insert into items (id, title, stock, supplier_days, supplier_id) values (1004, 'Small Pan Flute', 3, 7, 5001);

insert into items (id, title, stock, supplier_days, supplier_id) values (1005, 'Portal Gun', 0, 30, 5002);

CREATE OR REPLACE FUNCTION mysleep(integer) RETURNS integer
    AS 'select $1 from pg_sleep($1);'
    LANGUAGE SQL
    RETURNS NULL ON NULL INPUT;

CREATE OR REPLACE VIEW store (id, stock, supplier_days) AS select id, stock, supplier_days from items;
CREATE OR REPLACE VIEW store_slow (id, stock, supplier_days) AS select id, stock, supplier_days, mysleep(20) from items;
```

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
