# openshift-demo-helm-rabbitmq-db2

## Overview

This repository illustrates a reference implementation of Senzing using IBM's Db2 as the underlying database.

The instructions show how to set up a system that:

1. Reads JSON lines from a file on the internet.
1. Sends each JSON line to a message queue.
    1. In this implementation, the queue is RabbitMQ.
1. Reads messages from the queue and inserts into Senzing.
    1. In this implementation, Senzing keeps its data in an IBM Db2 database.
1. Reads information from Senzing via [Senzing REST API](https://github.com/Senzing/senzing-rest-api) server.
1. Views resolved entities in a [web app](https://github.com/Senzing/entity-search-web-app).

The following diagram shows the relationship of the Helm charts, docker containers, and code in this OpenShift demonstration.

![Image of architecture](architecture.png)

### Contents

1. [Expectations](#expectations)
    1. [Space](#space)
    1. [Time](#time)
    1. [Background knowledge](#background-knowledge)
1. [Prerequisites](#prerequisites)
    1. [Prerequisite software](#prerequisite-software)
    1. [Clone repository](#clone-repository)
1. [Demonstrate](#demonstrate)
    1. [EULA](#eula)
    1. [Set environment variables](#set-environment-variables)
    1. [Create custom helm values files](#create-custom-helm-values-files)
    1. [Create custom kubernetes configuration files](#create-custom-kubernetes-configuration-files)
    1. [Create namespace](#create-namespace)
    1. [Create persistent volume](#create-persistent-volume)
    1. [Add helm repositories](#add-helm-repositories)
    1. [Deploy Senzing RPM](#deploy-senzing-rpm)
    1. [Install IBM Db2 Driver](#install-ibm-db2-driver)
    1. [Install senzing-debug Helm chart](#install-senzing-debug-helm-chart)
    1. [Install DB2 Helm chart](#install-db2-helm-chart)
    1. [Install RabbitMQ Helm chart](#install-rabbitmq-helm-chart)
    1. [Install mock-data-generator Helm chart](#install-mock-data-generator-helm-chart)
    1. [Install init-container Helm chart](#install-init-container-helm-chart)
    1. [Install stream-loader Helm chart](#install-stream-loader-helm-chart)
    1. [Install senzing-api-server Helm chart](#install-senzing-api-server-helm-chart)
    1. [Install senzing-entity-search-web-app Helm chart](#install-senzing-entity-search-web-app-helm-chart)
    1. [View data](#view-data)
1. [Cleanup](#cleanup)
    1. [Delete everything in project](#delete-everything-in-project)
    1. [Delete minishift cluster](#delete-minishift-cluster)

## Expectations

### Space

This repository and demonstration require 20 GB free disk space.

### Time

Budget 4 hours to get the demonstration up-and-running, depending on CPU and network speeds.

### Background knowledge

This repository assumes a working knowledge of:

1. [Docker](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/docker.md)
1. [Openshift](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/openshift.md)
1. [Helm](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/helm.md)

## Prerequisites

### Prerequisite software

### minishift cluster

1. [Install minishift](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-minishift.md).
1. :pencil2: Set profile.
   Example:

    ```console
    export MY_MINISHIFT_PROFILE=minishift
    ```

1. Set profile parameter.
   Example:

    ```console
    export MY_MINISHIFT_PROFILE_PARAMETER="--profile ${MY_MINISHIFT_PROFILE}"
    ```

1. Enable addons.
   Example:

    ```console
    minishift addons install --defaults ${MY_MINISHIFT_PROFILE_PARAMETER}
    minishift addons enable admin-user ${MY_MINISHIFT_PROFILE_PARAMETER}
    ```

1. Start cluster.
   Example:

    ```console
    minishift start \
      --cpus 4 \
      --memory 10GB \
      --disk-size 150GB \
      --openshift-version v3.10.0 \
      ${MY_MINISHIFT_PROFILE_PARAMETER}
    ```

### Helm

1. [Install Helm](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-helm.md)
   on workstation.

1. FIXME:
   **May not be needed.**
   Set environment variables.
   Example:

    ```console
    eval "$(minishift oc-env)"

    export TILLER_HOST="$(minishift ip):$(oc get svc/tiller \
      -o jsonpath='{.spec.ports[0].nodePort}' \
      -n kube-system --as=system:admin)"

    export HELM_HOST="$(minishift ip):$(oc get svc/tiller \
      -o jsonpath='{.spec.ports[0].nodePort}' \
      -n kube-system --as=system:admin)"

    export MINISHIFT_ADMIN_CONTEXT="default/$(oc config view \
      -o jsonpath='{.contexts[?(@.name=="minishift")].context.cluster}')/system:admin"
    ```

### Clone repository

The Git repository has files that will be used in the `helm install --values` parameter.

1. Using these environment variable values:

    ```console
    export GIT_ACCOUNT=senzing
    export GIT_REPOSITORY=openshift-demo
    export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
    export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
    ```

1. Follow steps in [clone-repository](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/clone-repository.md) to install the Git repository.

## Demonstrate

### Log into OpenShift

1. :pencil2: Set profile.
   Example:

    ```console
    export MY_MINISHIFT_PROFILE=minishift
    ```

1. Set profile parameter.
   Example:

    ```console
    export MY_MINISHIFT_PROFILE_PARAMETER="--profile ${MY_MINISHIFT_PROFILE}"
    ```

1. Set path for `oc`.
   Example:

    ```console
    eval "$(minishift oc-env)"
    ```

1. Choose context.
   Example:

    ```console
    oc config use-context ${MY_MINISHIFT_PROFILE}
    ```

1. Login into OpenShift.
   Example:

    ```console
    oc login -u system:admin
    ```

    oc project

### Tiller

1. FIXME: Enable tiller addon.
   Example:

    ```console
    kubectl --namespace kube-system create sa tiller

    kubectl create clusterrolebinding tiller \
      --clusterrole cluster-admin \
      --serviceaccount=kube-system:tiller

    helm init --service-account tiller
    helm init --service-account tiller --upgrade
    ```

### EULA

To use the Senzing code, you must agree to the End User License Agreement (EULA).

1. :warning: This step is intentionally tricky and not simply copy/paste.
   This ensures that you make a conscious effort to accept the EULA.
   Example:

    <code>export SENZING_ACCEPT_EULA="&lt;the value from [this link](https://github.com/Senzing/knowledge-base/blob/master/lists/environment-variables.md#senzing_accept_eula)&gt;"</code>

### Environment variables

1. Set environment variables listed in "[Clone repository](#clone-repository)".

1. :pencil2: Environment variables that need customization.
   Example:

    ```console
    export DEMO_PREFIX=my
    export DEMO_NAMESPACE=${DEMO_PREFIX}-namespace

    export DOCKER_REGISTRY_URL=docker.io
    export DOCKER_REGISTRY_SECRET=${DOCKER_REGISTRY_URL}-secret
    ```

### Security context

1. :pencil2: Environment variables for `securityContext` values.
   Example:

    ```console
    export SENZING_RUN_AS_USER=1001
    export SENZING_RUN_AS_GROUP=1001
    export SENZING_FS_GROUP=1001
    ```

### Create custom helm values files

:thinking: In this step, Helm template files are populated with actual values.
There are two methods of accomplishing this.
Only one method needs to be performed.

1. **Method #1:** Quick method using `envsubst`.
   Example:

    ```console
    export HELM_VALUES_DIR=${GIT_REPOSITORY_DIR}/helm-values
    mkdir -p ${HELM_VALUES_DIR}

    for file in ${GIT_REPOSITORY_DIR}/helm-values-templates/*.yaml; \
    do \
      envsubst < "${file}" > "${HELM_VALUES_DIR}/$(basename ${file})";
    done
    ```

1. **Method #2:** Copy and manually modify files method.
   Example:

    ```console
    export HELM_VALUES_DIR=${GIT_REPOSITORY_DIR}/helm-values
    mkdir -p ${HELM_VALUES_DIR}

    cp ${GIT_REPOSITORY_DIR}/helm-values-templates/* ${HELM_VALUES_DIR}
    ```

    :pencil2: Edit files in ${HELM_VALUES_DIR} replacing the following variables with actual values.

    1. `${DEMO_PREFIX}`
    1. `${DOCKER_REGISTRY_SECRET}`
    1. `${DOCKER_REGISTRY_URL}`
    1. `${SENZING_ACCEPT_EULA}`
    1. `${SENZING_DATABASE_URL}`
    1. `${SENZING_FS_GROUP}`
    1. `${SENZING_RUN_AS_USER}`
    1. `${SENZING_RUN_AS_GROUP}`

### Create custom kubernetes configuration files

:thinking: In this step, Kubernetes template files are populated with actual values.
There are two methods of accomplishing this.
Only one method needs to be performed.

1. **Method #1:** Quick method using `envsubst`.
   Example:

    ```console
    export KUBERNETES_DIR=${GIT_REPOSITORY_DIR}/kubernetes
    mkdir -p ${KUBERNETES_DIR}

    for file in ${GIT_REPOSITORY_DIR}/kubernetes-templates/*; \
    do \
      envsubst < "${file}" > "${KUBERNETES_DIR}/$(basename ${file})";
    done
    ```

1. **Method #2:** Copy and manually modify files method.
   Example:

    ```console
    export KUBERNETES_DIR=${GIT_REPOSITORY_DIR}/kubernetes
    mkdir -p ${KUBERNETES_DIR}

    cp ${GIT_REPOSITORY_DIR}/kubernetes-templates/* ${KUBERNETES_DIR}
    ```

    :pencil2: Edit files in ${KUBERNETES_DIR} replacing the following variables with actual values.

    1. `${DEMO_NAMESPACE}`

### Create OpenShift project

1. :pencil2: Set environment variables.
   Example:

    ```console
    export OC_DESCRIPTION="My description..."
    export OC_DISPLAY_NAME="My Senzing project"
    ```

1. Create project.
   Example:

    ```console
    oc new-project ${DEMO_NAMESPACE} \
      --description="${OC_DESCRIPTION}" \
      --display-name="${OC_DISPLAY_NAME}"
    ```

1. XXXX
   Example:

    ```console
    oc project ${DEMO_NAMESPACE}
    ```

### Create persistent volume

Minishift creates persistent volumes automatically.

1. :thinking: **Optional:** Review persistent volumes.
   Example:

    ```console
    oc get persistentvolumes \
      --namespace ${DEMO_NAMESPACE}
    ```

### Create persistent volume claims

1. Create persistent volume claims.
   Example:

    ```console
    oc create -f ${KUBERNETES_DIR}/persistent-volume-claim-db2.yaml
    oc create -f ${KUBERNETES_DIR}/persistent-volume-claim-rabbitmq.yaml
    oc create -f ${KUBERNETES_DIR}/persistent-volume-claim-senzing.yaml
    ```

1. :thinking: **Optional:** Review persistent volumes and claims.
   Example:

    ```console
    oc get persistentvolumeClaims \
      --namespace ${DEMO_NAMESPACE}
    ```

### Create Service Context Constraint

1. Create Security Constraint Context.
   Example:

    ```console
    oc create -f ${KUBERNETES_DIR}/security-context-constraint-runasany.yaml
    oc create -f ${KUBERNETES_DIR}/security-context-constraint-limited.yaml
    ```

1. :thinking: **Optional:** Review persistent volumes and claims.
   Example:

    ```console
    oc get securityContextConstraints \
      --namespace ${DEMO_NAMESPACE}
    ```

### Initialize Helm

1. Connect Helm to MiniShift.
   Example:

    ```console
    helm init
    ```

### Add helm repositories

1. Add Senzing repository.
   Example:

    ```console
    helm repo add senzing https://senzing.github.io/charts/
    ```

1. Update repositories.
   Example:

    ```console
    helm repo update
    ```

1. :thinking: **Optional:**
   Review repositories.
   Example:

    ```console
    helm repo list
    ```

1. Reference: [helm repo](https://helm.sh/docs/helm/#helm-repo)

### Deploy Senzing RPM

This deployment initializes the Persistent Volume with Senzing code and data.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-runasany \
      -z ${DEMO_PREFIX}-senzing-yum
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-yum \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-yum.yaml \
      senzing/senzing-yum
    ```

### Install IBM Db2 Driver

This deployment adds the IBM Db2 Client driver code to the Persistent Volume.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-runasany \
      -z ${DEMO_PREFIX}-ibm-db2-driver-installer
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-ibm-db2-driver-installer \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/ibm-db2-driver-installer.yaml \
      senzing/ibm-db2-driver-installer
    ```

### Install Db2 Helm chart

This step starts IBM Db2 database and populates the database with the Senzing schema.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-runasany \
      -z ${DEMO_PREFIX}-senzing-ibm-db2
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-ibm-db2 \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-ibm-db2.yaml \
      senzing/senzing-ibm-db2
    ```

### Install RabbitMQ Helm chart

This deployment creates a RabbitMQ service.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-runasany \
      -z ${DEMO_PREFIX}-rabbitmq
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-rabbitmq \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/rabbitmq.yaml \
      stable/rabbitmq
    ```

1. Wait for pods to run.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. :thinking: **Optional:** To view RabbitMQ, see [View RabbitMQ](#view-rabbitmq)

### Install senzing-mock-data-generator Helm chart

The mock data generator pulls JSON lines from a file and pushes them to RabbitMQ.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-limited \
      -z ${DEMO_PREFIX}-senzing-mock-data-generator
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-mock-data-generator \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-mock-data-generator-rabbitmq.yaml \
      senzing/senzing-mock-data-generator
    ```

### Install init-container Helm chart

The init-container creates files from templates and initializes the G2 database.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-runasany \
      -z ${DEMO_PREFIX}-senzing-init-container
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-init-container \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-init-container-db2.yaml \
      senzing/senzing-init-container
    ```

### Install senzing-stream-loader Helm chart

The stream loader pulls messages from RabbitMQ and sends them to Senzing.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-limited \
      -z ${DEMO_PREFIX}-senzing-stream-loader
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-stream-loader \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-stream-loader-rabbitmq-db2.yaml \
      senzing/senzing-stream-loader
    ```

### Install senzing-api-server Helm chart

The Senzing API server receives HTTP requests to read and modify Senzing data.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-limited \
      -z ${DEMO_PREFIX}-senzing-api-server
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-api-server \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-api-server.yaml \
      senzing/senzing-api-server
    ```

1. Wait for pods to run.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. :thinking: **Optional:** To view Senzing API server, see [View Senzing API Server](#view-senzing-api-server).

### Install senzing-entity-search-web-app Helm chart

The Senzing Entity Search WebApp is a light-weight WebApp demonstrating Senzing search capabilities.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-limited \
      -z ${DEMO_PREFIX}-senzing-entity-search-web-app
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-entity-search-web-app \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-entity-search-web-app.yaml \
      senzing/senzing-entity-search-web-app
    ```

1. Wait for pod to run.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. :thinking: **Optional:** To view Senzing Entity Search WebApp, see [View Senzing Entity Search WebApp](#view-senzing-entity-search-webapp).

### Optional charts

These charts are not necessary for the demonstration,
but may be valuable in a production environment.

#### Install senzing-base Helm Chart

This deployment provides a pod that is used to copy files to and from the Persistent Volume
in later steps.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-runasany \
      -z ${DEMO_PREFIX}-senzing-base
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-base \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-base.yaml \
       senzing/senzing-base
    ```

1. Find pod name.
   Example:

    ```console
    export SENZING_BASE_POD_NAME=$(oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --output jsonpath="{.items[0].metadata.name}" \
      --selector "app.kubernetes.io/name=senzing-base, \
                  app.kubernetes.io/instance=${DEMO_PREFIX}-senzing-base" \
      )
    ```

1. Wait for pods to run.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

### Install senzing-redoer Helm chart

The Senzing Redoer processes Senzing "redo" records.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-limited \
      -z ${DEMO_PREFIX}-senzing-redoer
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-redoer \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-redoer-db2.yaml \
      senzing/senzing-redoer
    ```

#### Install senzing-configurator Helm chart

The Senzing Configurator is a micro-service for changing Senzing configuration.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-limited \
      -z ${DEMO_PREFIX}-senzing-configurator
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-configurator \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-configurator-db2.yaml \
      senzing/senzing-configurator
    ```

1. :thinking: **Optional:** To view Senzing Configurator, see [View Senzing Configurator](#view-senzing-configurator).

### View data

1. Username and password for the following sites are the values seen in the corresponding "values" YAML file located in
   [helm-values-templates](../../helm-values-templates).

#### Update hosts file

The `/etc/hosts` file needs to be updated with a line like:

```console
10.10.10.10 rabbitmq.local senzing-api.local senzing-configurator.local senzing-entity-search.local
```

:thinking:  Instead of `10.10.10.10`, the real IP address needs to be found.
There are 2 methods to find the IP address.

1. **Method #1:** Ping the "infra node".
   Example:

    1. Determine the IP address of the OpenShift "infra" node.
       Example:

        ```console
        export SENZING_INFRA_NODE=$(oc get nodes \
          --output jsonpath="{.items[0].metadata.name}" \
          --selector "node-role.kubernetes.io/infra=true" \
        )
        ping ${SENZING_INFRA_NODE}
        ```

    1. From out output of the `ping` command, the IP address can be found.

1. **Method #2:** Extract the IP Address.
   Example:

    1. Query the IP address directly.
       Example:

        ```console
        export SENZING_INFRA_NODE_IP_ADDRESS=$(oc get nodes \
          --output jsonpath="{.items[0].status.addresses[0].address}" \
          --selector "node-role.kubernetes.io/infra=true" \
        )
        echo ${SENZING_INFRA_NODE_IP_ADDRESS}
        ```

1. :pencil2: Into the `/etc/hosts` file, append a line like the following example, replacing `10.10.10.10` with the infra node IP address.

    ```console
    10.10.10.10 rabbitmq.local senzing-api.local senzing-configurator.local senzing-entity-search.local
    ```

#### View RabbitMQ

1. If not already done, [update hosts file](#update-hosts-file).
1. RabbitMQ will be viewable at [rabbitmq.local](http://rabbitmq.local).
    1. Login
        1. See [helm-values/rabbitmq.yaml](../../helm-values/rabbitmq.yaml) for Username and password.
        1. Default: user/passw0rd (seen in [helm-values-templates/rabbitmq.yaml](../../helm-values-templates/rabbitmq.yaml))

#### View Senzing Configurator

1. If not already done, [update hosts file](#update-hosts-file).
1. Senzing Configurator will be viewable at [senzing-configurator.local/datasources](http://senzing-configurator.local/datasources).
1. Make HTTP calls via `curl`.
   Example:

    ```console
    export SENZING_CONFIGURATOR_SERVICE=http://senzing-configurator.local

    curl -X GET ${SENZING_CONFIGURATOR_SERVICE}/datasources
    curl -X POST \
      --data '[ "TEST", "TEST1", "TEST2", "TEST3"]' \
      --header 'Content-type: application/json;charset=utf-8' \
      ${SENZING_CONFIGURATOR_SERVICE}/datasources
    ```

#### View Senzing API Server

1. If not already done, [update hosts file](#update-hosts-file).
1. View results from Senzing REST API server.
   The server supports the
   [Senzing REST API](https://github.com/Senzing/senzing-rest-api).
   1. From a web browser.
      Examples:
      1. [senzing-api.local/heartbeat](http://senzing-api.local/heartbeat)
      1. [senzing-api.local/license](http://senzing-api.local/license)
      1. [senzing-api.local/entities/1](http://senzing-api.local/entities/1)
   1. From `curl`.
      Examples:

        ```console
        export SENZING_API_SERVICE=http://senzing-api.local

        curl -X GET ${SENZING_API_SERVICE}/heartbeat
        curl -X GET ${SENZING_API_SERVICE}/license
        curl -X GET ${SENZING_API_SERVICE}/entities/1
        ```

   1. From [OpenApi "Swagger" editor](http://editor.swagger.io/?url=https://raw.githubusercontent.com/Senzing/senzing-rest-api/master/senzing-rest-api.yaml).

#### View Senzing Entity Search WebApp

1. If not already done, [update hosts file](#update-hosts-file).
1. Senzing Entity Search WebApp will be viewable at [senzing-entity-search.local](http://senzing-entity-search.local).
   The [demonstration](https://github.com/Senzing/knowledge-base/blob/master/demonstrations/docker-compose-web-app.md)
   instructions will give a tour of the Senzing web app.

## Troubleshooting

### Install senzing-debug Helm chart

This deployment provides a pod that can be used to view Persistent Volumes
and run Senzing utility programs.

1. Add Security Context Constraint.
   Example:

    ```console
    oc adm policy add-scc-to-user \
      senzing-security-context-constraint-runasany \
      -z ${DEMO_PREFIX}-senzing-debug
    ```

1. Install chart.
   Example:

    ```console
    helm install \
      --name ${DEMO_PREFIX}-senzing-debug \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-debug.yaml \
       senzing/senzing-debug
    ```

1. Wait for pod to run.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. Find pod name.
   Example:

    ```console
    export SENZING_DEBUG_POD_NAME=$(oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --output jsonpath="{.items[0].metadata.name}" \
      --selector "app.kubernetes.io/name=senzing-debug, \
                  app.kubernetes.io/instance=${DEMO_PREFIX}-senzing-debug" \
      )
    ```

1. Log into debug pod.
   Example:

    ```console
    oc exec -it --namespace ${DEMO_NAMESPACE} ${SENZING_DEBUG_POD_NAME} -- /bin/bash
    ```

### Support

Additional information:

1. [Helm Charts](https://github.com/Senzing/awesome#helm-charts)
1. [Docker images on Docker Hub](https://github.com/Senzing/awesome#dockerhub)
1. [Dockerfiles](https://github.com/Senzing/awesome#dockerfiles)

If the instructions don't address an issue you are seeing, please "submit a request" so we can help you.

1. [Submit a request](https://senzing.zendesk.com/hc/en-us/requests/new)
1. Email: [support@senzing.com](mailto:support@senzing.com)
1. [Report an issue on GitHub](https://github.com/Senzing/ibm-openshift-guide/issues)

This repository is a community project.
Feel free to submit a Pull Request for change.

## Cleanup

### Delete everything in project

1. Example:

    ```console
    helm delete --purge ${DEMO_PREFIX}-senzing-entity-search-web-app
    helm delete --purge ${DEMO_PREFIX}-senzing-api-server
    helm delete --purge ${DEMO_PREFIX}-senzing-redoer
    helm delete --purge ${DEMO_PREFIX}-senzing-stream-loader
    helm delete --purge ${DEMO_PREFIX}-senzing-configurator
    helm delete --purge ${DEMO_PREFIX}-senzing-init-container
    helm delete --purge ${DEMO_PREFIX}-senzing-base
    helm delete --purge ${DEMO_PREFIX}-senzing-ibm-db2
    helm delete --purge ${DEMO_PREFIX}-ibm-db2-driver-installer
    helm delete --purge ${DEMO_PREFIX}-senzing-yum
    helm delete --purge ${DEMO_PREFIX}-senzing-mock-data-generator
    helm delete --purge ${DEMO_PREFIX}-rabbitmq
    helm delete --purge ${DEMO_PREFIX}-senzing-debug
    helm repo remove senzing
    oc delete -f ${KUBERNETES_DIR}/security-context-constraint-limited.yaml
    oc delete -f ${KUBERNETES_DIR}/security-context-constraint-runasany.yaml
    oc delete -f ${KUBERNETES_DIR}/persistent-volume-claim-db2.yaml
    oc delete -f ${KUBERNETES_DIR}/persistent-volume-claim-senzing.yaml
    oc delete -f ${KUBERNETES_DIR}/persistent-volume-claim-rabbitmq.yaml
    ```

### Delete minikube cluster

1. Example:

    ```console
    minishift stop
    minishift delete --force --clear-cache
    ```

### Delete git repository

1. Delete git repository.  Example:

    ```console
    sudo rm -rf ${GIT_REPOSITORY_DIR}
    ```

## References

1. [OKD](https://docs.okd.io/)
    1. [minishift](https://docs.okd.io/latest/minishift/index.html)
        1. [Minishift basic usage](https://docs.okd.io/latest/minishift/using/basic-usage.html)
1. [github.com/minishift/minishift](https://github.com/minishift/minishift/)
    1. [github.com/minishift/minishift-addons/helm](https://github.com/minishift/minishift-addons/tree/master/add-ons/helm)
1. [Adding Persistent Storage to Minishift / CDK 3 in Minutes](https://developers.redhat.com/blog/2017/04/05/adding-persistent-storage-to-minishift-cdk-3-in-minutes/)
