# [Beta] Invenio Helm Chart

This repository contains the helm chart to deploy an Invenio instance.

> :warning: Please note that this is a work in progress, the configuration might not suit a production deployment.

## Pre-requisites

- [OpenShift Client](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html#cli-installing-cli_cli-developer-commands)
- [Helm, version 3.x](https://helm.sh/docs/intro/install/)

## Usage

> :warning: Currently this helm charts are only compatible with OpenShift.

### OpenShift

Login and select the right project:

```console
$ oc login <your.openshift.cluster>
$ oc project invenio
```

Create all the needed secrets and install Invenio.

#### 1. Secrets

Database secrets:

```console
$ POSTGRESQL_PASSWORD=$(openssl rand -hex 8)
$ POSTGRESQL_USER=invenio
$ POSTGRESQL_HOST=db
$ POSTGRESQL_PORT=5432
$ POSTGRESQL_DATABASE=invenio
$ oc create secret generic \
  --from-literal="POSTGRESQL_PASSWORD=$POSTGRESQL_PASSWORD" \
  --from-literal="SQLALCHEMY_DB_URI=postgresql+psycopg2://$POSTGRESQL_USER:$POSTGRESQL_PASSWORD@$POSTGRESQL_HOST:$POSTGRESQL_PORT/$POSTGRESQL_DATABASE" \
  db-secrets
secret "db-secrets" created
```

RabbitMQ secrets:

```console
$ RABBITMQ_DEFAULT_PASS=$(openssl rand -hex 8)
$ oc create secret generic \
  --from-literal="RABBITMQ_DEFAULT_PASS=$RABBITMQ_DEFAULT_PASS" \
  --from-literal="CELERY_BROKER_URL=amqp://guest:$RABBITMQ_DEFAULT_PASS@mq:5672/" \
  mq-secrets
secret "mq-secrets" created
```

Elasticsearch secrets:

```console
$ ELASTICSEARCH_PASSWORD=$(openssl rand -hex 8)
$ ELASTICSEARCH_USER=username
$ oc create secret generic \
  --from-literal="ELASTICSEARCH_PASSWORD=$ELASTICSEARCH_PASSWORD" \
  --from-literal="ELASTICSEARCH_USER=$ELASTICSEARCH_USER" \
  elasticsearch-secrets
```

> :warning: For now the previous Elasticsearch environment variables are not ported to `invenio-search`, therefore the way to create the secret is:

``` console
$ export INVENIO_SEARCH_ELASTIC_HOSTS="[{'host': 'es-cdsbooks.cern.ch', 'url_prefix': '/es', 'timeout': 30, 'port': 443, 'use_ssl': True, 'verify_certs': False, 'http_auth':('USERNAME_CHANGEME', 'PASSWORD_CHANGEME')}]"
$ oc create secret generic \
  --from-literal="INVENIO_SEARCH_ELASTIC_HOSTS=$INVENIO_SEARCH_ELASTIC_HOSTS" \
  elasticsearch-secrets
```

#### 2. Install Invenio

```console
$ helm install invenio ./invenio --disable-openapi-validation
NAME: invenio
LAST DEPLOYED: Mon Mar  9 16:25:15 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:Invenio is ready to rock :rocket:
```

> :warning: Passing `--disable-openapi-validation` as there is currently a problem with OpenShift objects and Helm when it comes to client side validation, see [issue](https://github.com/openshift/origin/issues/24060).

#### 3. Setup the instance:

Get a bash terminal in a pod:

```console
$ oc get pods
$ oc exec -it <web-pod> bash
```

Setup the instance:

``` console
$ . scl_source enable rh-python36
$ invenio db init # If the db does not exist already
$ invenio db create
$ invenio index init
$ invenio index queue init purge
$ invenio files location --default 'default-location'  $(invenio shell --no-term-title -c "print(app.instance_path)")'/data'
$ invenio roles create admin
$ invenio access allow superuser-access role admin
```

#### 4. Launching jobs

**One time job**

``` console
$ oc process -f job.yml --param JOB_NAME='demo-data-1' \
  --param JOB_COMMAND='invenio demo create 300 1000' | oc create -f -
```

**Cron job**

``` console
$ oc process -f cronjob.yml --param JOB_NAME=index-run \
  --param JOB_COMMAND=invenio index run -d | oc create -f -
```
