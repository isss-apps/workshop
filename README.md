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

## The Integration

We will be responsible for an integration microservice, which is part of an "online store".

Your service will integrate the following services:

* *Catalog service* which maintains the product tree, mapping of catalogue (client-visible) ids to internal storeIds and item pricing.
* *Store DB* an SQL server with stock and supplier information
* *Ordering service* which handls the actual orders

to expose the following endpoints:

#### GET /catalog/list/${id}

Asking the catalog service to get the list of sub-categories or products in the given tree, filtering out the internal details (such as the storeId)

```
$ curl -q http://store.foobarter.org/catalog/list | python -m json.tool
[
    {
        "dir": true,
        "id": 0,
        "name": "Plush Toys",
        "price": null,
        "rootCategory": 0
    },
    {
        "dir": true,
        "id": 1,
        "name": "Instruments",
        "price": null,
        "rootCategory": 1
    },
    {
        "dir": true,
        "id": 2,
        "name": "Family Fun",
        "price": null,
        "rootCategory": 2
    }
]
```

```
$ curl -q http://store.foobarter.org/catalog/list/2 | python -m json.tool
[
    {
        "dir": false,
        "id": 8,
        "name": "Portal Gun",
        "price": 999.99,
        "rootCategory": 2
    }
]
```

#### GET /availability/${id}

Returns the availability information about the product. Asks the catalog service for the storeId, and does the SQL query to the Store DB to retreive the stock and supplier delay.

```
$ curl -q http://store.foobarter.org/availability/8
{"itemId":8,"inStock":false,"supplierDays":30}
```

#### PUT /order

Using the catalog service, translates the client order request to the order request the Ordering service understands.

```
$ cat order.json
{
  "name": "Random F. Flyer",
  "address": "Milliways, 25, MG, MW",
  "items": [
    {
      "catalogId": 3,
      "amount": 2
    },
    {
      "catalogId": 4,
      "amount": 1
    }
  ]
}

$ curl -X PUT -d @order.json http://store.foobarter.org/order
{"message":"Thank you Random F. Flyer for your order!","price":69.97,"duplicate":false}
```

### Task 00 - Wiring up the Camel Routes

First, checkout the project and the initial workshop tag, isss-00

```
git clone https://github.com/isss-apps/store.git
cd store
git checkout isss-00
```

and open the project in your favourite editor / IDE

#### Assignment: fill in the TODOs to wire up the routes

Edit src/main/java/org/foobarter/isss/store/StoreRoute.java

To build and run, use maven:

```
mvn -s configuration/settings.xml clean package

# Configure the DB username / password
export SPRING_DATASOURCE_USERNAME=student
export SPRING_DATASOURCE_PASSWORD=isss-secret

mvn -s configuration/settings.xml spring-boot:run
```

The store endpoints should be accessible on http://127.0.0.1:8080/

### Task 01 - Add Timeout to the Availability SQL Query

The database may be a bit overloaded at times, and since the information it provides is not essential, we should not block users when the query takes too long.

#### Assignment: Add and handle a query timeout of 5 seconds in the sql: query

In case of timeout, let the availability query return an HTTP 503 code with a nice custom nice "sorry" message in the body.

See 

* http://camel.apache.org/sql-component.html 
* http://docs.spring.io/spring/docs/2.5.x/javadoc-api/org/springframework/jdbc/core/JdbcTemplate.html
* http://camel.apache.org/exception-clause.html

### Task 02 - Add Circuit Breaker to the Availability SQL Query

While timeouts are better than infinite hangs, they still annoy users. Lets add a cricuit breaker, so that when the database takes too long, we just stop asking it for several minutes and fail immediately.

#### Assignment: Add a separate direct:storedb-query-with-circuit-breaker route in between direct:availability and direct:storedb-query

See

* http://camel.apache.org/load-balancer.html
* http://camel.apache.org/try-catch-finally.html  (optionally)

## Deploying to OpenShift

### Deploying the Store application

Create the build config and an image stream for our store image (named store)

```
oc new-build --binary=true -i openshift/fis-java-openshift:2.0 --name=store
```

Create the deployment config from our store image stream

```
oc new-app -i store --allow-missing-imagestream-tags
```

Build the image using locally built jar file

```
oc start-build store --from-file=target/store.jar
```

Now the application should be running, 

```
oc get pods
```

We can watch the pod logs

```
oc logs store-1-ok3ix -f
```

### Exposing the 8080 port to the outside

First we create the service (we expose the pod port 8080 as a kubernetes service)

```
oc expose dc store --port=8080
```

Kubernetes services are cluster-local by default,  we expose the kubernetes service also as an OpenShift Route, so it can be accessed from outside the kubernetes cluster

```
oc expose service store --hostname=store.`minishift ip`.nip.io
```

### Configuring pods via environment variables

The store should be working, except the availability DB, as it lacks the DB username/password

```
oc env dc store SPRING_DATASOURCE_USERNAME=student SPRING_DATASOURCE_PASSWORD=isss-secret
```

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
CREATE OR REPLACE VIEW store_slow (id, stock, supplier_days) AS select id, stock, supplier_days, mysleep(1) from items;

GRANT SELECT ON store_slow TO student;
GRANT SELECT ON store TO student;
```

Now you may edit the store DeploymentConfig to use the cluster-local DB service

```
oc env dc store SPRING_DATASOURCE_URL=jdbc:postgresql://postgresql/isss
```

## cicd-pipeline
Demo to introduce CI/CD pipelines on OpenShift.

### Instructions

Create necessary resources on OpenShift instance.

```
oc create -f fis-image-streams.json -n openshift

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
