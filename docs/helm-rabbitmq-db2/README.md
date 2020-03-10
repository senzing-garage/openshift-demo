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
    1. [Minishift](#minishift)
    1. [Helm](#helm)
    1. [Clone repository](#clone-repository)
1. [Demonstrate](#demonstrate)
    1. [Set environment variables](#set-environment-variables)
    1. [EULA](#eula)
    1. [Create Openshift cluster](#create-openshift-cluster)
    1. [Log into OpenShift](#log-into-openshift)
    1. [Create custom helm values files](#create-custom-helm-values-files)
    1. [Create custom kubernetes configuration files](#create-custom-kubernetes-configuration-files)
    1. [Create OpenShift project](#create-openshift-project)
    1. [Create persistent volume](#create-persistent-volume)
    1. [Create persistent volume claims](#create-persistent-volume-claims)
    1. [Create service context constraint](#create-service-context-constraint)
    1. [Initialize Helm and Tiller](#initialize-helm-and-tiller)
    1. [Add helm repositories](#add-helm-repositories)
    1. [Deploy Senzing RPM](#deploy-senzing-rpm)
    1. [Install IBM Db2 Driver](#install-ibm-db2-driver)
    1. [Install RabbitMQ Helm chart](#install-rabbitmq-helm-chart)
    1. [Install DB2 Helm chart](#install-db2-helm-chart)
    1. [Install senzing-mock-data-generator Helm chart](#install-senzing-mock-data-generator-helm-chart)
    1. [Install init-container Helm chart](#install-init-container-helm-chart)
    1. [Install senzing-stream-loader Helm chart](#install-senzing-stream-loader-helm-chart)
    1. [Install senzing-api-server Helm chart](#install-senzing-api-server-helm-chart)
    1. [Install senzing-entity-search-web-app Helm chart](#install-senzing-entity-search-web-app-helm-chart)
    1. [Optional charts](#optional-charts)
        1. [Install senzing-debug Helm Chart](#install-senzing-debug-helm-chart)
        1. [Install senzing-redoer Helm chart](#install-senzing-redoer-helm-chart)
        1. [Install senzing-configurator Helm chart](#install-senzing-configurator-helm-chart)
    1. [View data](#view-data)
        1. [View RabbitMQ](#view-rabbitmq)
        1. [View Senzing API Server](#view-senzing-api-server)
        1. [View Senzing Entity Search WebApp](#view-senzing-entity-search-webapp)
1. [Cleanup](#cleanup)
    1. [Delete everything in project](#delete-everything-in-project)
    1. [Delete minishift cluster](#delete-minishift-cluster)
    1. [Delete git repository](#delete-git-repository)
1. [Support](#support)
1. [References](#references)

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

### Minishift

1. [Install minishift](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-minishift.md).
    1. Instructions tested with minishift version 1.34.2.

### Helm

1. [Install Helm](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-helm.md)
   on workstation.
    1. Instructions are written for Helm version 2.x.
       The instructions do not work with Helm version 3.x.

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

### Set environment variables

1. Set environment variables listed in "[Clone repository](#clone-repository)".

    ```console
    export GIT_ACCOUNT=senzing
    export GIT_REPOSITORY=openshift-demo
    export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
    export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
    ```

1. Set environment variables for minishift.
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

1. :pencil2: Environment variables that need customization.
   Example:

    ```console
    export DEMO_PREFIX=my
    export DEMO_NAMESPACE=${DEMO_PREFIX}-namespace

    export DOCKER_REGISTRY_URL=docker.io
    export DOCKER_REGISTRY_SECRET=${DOCKER_REGISTRY_URL}-secret
    ```

1. :thinking: **Optional:** If using an insecure docker registry,
   set the following environment variable.
   Example:

    ```console
    export MINISHIFT_INSECURE_REGISTRY_PARAMETER="--insecure-registry ${DOCKER_REGISTRY_URL}"
    ```

1. :pencil2: Environment variables for `securityContext` values.
   Example:

    ```console
    export SENZING_RUN_AS_USER=1001
    export SENZING_RUN_AS_GROUP=1001
    export SENZING_FS_GROUP=1001
    ```

1. Directories.
   Example:

    ```console
    export HELM_VALUES_DIR=${GIT_REPOSITORY_DIR}/helm-values
    export KUBERNETES_DIR=${GIT_REPOSITORY_DIR}/kubernetes
    ```

1. :pencil2: `oc` project metadata.
   Example:

    ```console
    export OC_DESCRIPTION="My Senzing description..."
    export OC_DISPLAY_NAME="My Senzing project"
    ```

### EULA

To use the Senzing code, you must agree to the End User License Agreement (EULA).

1. :warning: This step is intentionally tricky and not simply copy/paste.
   This ensures that you make a conscious effort to accept the EULA.
   Example:

    <code>export SENZING_ACCEPT_EULA="&lt;the value from [this link](https://github.com/Senzing/knowledge-base/blob/master/lists/environment-variables.md#senzing_accept_eula)&gt;"</code>

### Create OpenShift cluster

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
      ${MINISHIFT_INSECURE_REGISTRY_PARAMETER} \
      ${MY_MINISHIFT_PROFILE_PARAMETER}
    ```

1. :thinking: **Optional:** To view OpenShift console, see [View OpenShift console](#view-openshift-console)

### Log into OpenShift

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

### Create custom helm values files

:thinking: In this step, Helm template files are populated with actual values.
There are two methods of accomplishing this.
Only one method needs to be performed.

1. **Method #1:** Quick method using `envsubst`.
   Example:

    ```console
    mkdir -p ${HELM_VALUES_DIR}
    for file in ${GIT_REPOSITORY_DIR}/helm-values-templates/*.yaml; \
    do \
      envsubst < "${file}" > "${HELM_VALUES_DIR}/$(basename ${file})";
    done
    ```

1. **Method #2:** Copy and manually modify files method.
   Example:

    ```console
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
    mkdir -p ${KUBERNETES_DIR}
    for file in ${GIT_REPOSITORY_DIR}/kubernetes-templates/*; \
    do \
      envsubst < "${file}" > "${KUBERNETES_DIR}/$(basename ${file})";
    done
    ```

1. **Method #2:** Copy and manually modify files method.
   Example:

    ```console
    mkdir -p ${KUBERNETES_DIR}
    cp ${GIT_REPOSITORY_DIR}/kubernetes-templates/* ${KUBERNETES_DIR}
    ```

    :pencil2: Edit files in ${KUBERNETES_DIR} replacing the following variables with actual values.

    1. `${DEMO_NAMESPACE}`

### Create OpenShift project

1. Create project.
   Example:

    ```console
    oc new-project ${DEMO_NAMESPACE} \
      --description="${OC_DESCRIPTION}" \
      --display-name="${OC_DISPLAY_NAME}"
    ```

1. Switch to newly created project.
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

### Create service context constraint

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

### Initialize Helm and Tiller

1. Enable tiller addon.
   Example:

    ```console
    oc create serviceaccount tiller \
      --namespace kube-system

    oc create clusterrolebinding tiller \
      --clusterrole cluster-admin \
      --serviceaccount=kube-system:tiller

    helm init --service-account tiller
    helm init --service-account tiller --upgrade
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

1. :thinking: **Optional:** Review repositories.
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

1. Wait for pods to complete.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
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

1. :thinking: **Optional:** To view RabbitMQ, see [View RabbitMQ](#view-rabbitmq)

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

1. Wait for pods to run.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. Wait for Db2 schema creation to complete.
   Example:

    ```console
    export DB2_POD_NAME=$(oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --output jsonpath="{.items[0].metadata.name}" \
      --selector "app.kubernetes.io/name=senzing-ibm-db2, \
                  app.kubernetes.io/instance=${DEMO_PREFIX}-senzing-ibm-db2" \
    )

    oc logs --namespace ${DEMO_NAMESPACE} ${DB2_POD_NAME}
    ```

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

1. Wait for pods to run.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
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

1. Wait for pods to run.
   Example:

    ```console
    oc get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
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

#### Install senzing-debug Helm Chart

This deployment provides a pod that is used for debugging.

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
    oc exec \
      --tty \
      --stdin \
      --namespace ${DEMO_NAMESPACE} \
      ${SENZING_DEBUG_POD_NAME} \
      -- /bin/bash
    ```

#### Install senzing-redoer Helm chart

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

#### View OpenShift console

1. Launch OpenShift in default browser.
   Example:

    ```console
    minishift console
    ```

1. :thinking: **Alternative:**  Locate OpenShift console URL.
   Example:

    ```console
    minishift console --url
    ```

#### Modify hosts file

1. Backup `/etc/hosts`
   Example:

    ```console
    sudo cp /etc/hosts /etc/hosts.$(date +%s)
    ```

1. Append line to `/etc/hosts`.
   Example:

    ```console
    echo "$(minishift ip) rabbitmq.local senzing-entity-search.local senzing-api.local senzing-configurator.local" \
    | sudo tee -a /etc/hosts
    ```

#### View RabbitMQ

1. If not done previously, [modify hosts file](#modify-hosts-file)
1. RabbitMQ is viewable at
   [http://rabbitmq.local](http://rabbitmq.local).
    1. **Defaults:** username: `user` password: `passw0rd`
1. See
   [additional tips](https://github.com/Senzing/knowledge-base/blob/master/lists/docker-compose-demo-tips.md#rabbitmq)
   for working with RabbitMQ.

#### View Senzing API Server

View results from Senzing REST API server.
The server supports the
[Senzing REST API](https://github.com/Senzing/senzing-rest-api).

1. If not done previously, [modify hosts file](#modify-hosts-file)
1. View REST API using [OpenApi "Swagger" editor](http://editor.swagger.io/?url=https://raw.githubusercontent.com/Senzing/senzing-rest-api/master/senzing-rest-api.yaml).
    1. In **Servers**, choose `http://senzing-api.local`
1. Example Senzing REST API request:
   [http://senzing-api.local//heartbeat](http://senzing-api.local/heartbeat)
1. See
   [additional tips](https://github.com/Senzing/knowledge-base/blob/master/lists/docker-compose-demo-tips.md#senzing-api-server)
   for working with Senzing API server.

#### View Senzing Entity Search WebApp

1. If not done previously, [modify hosts file](#modify-hosts-file)
1. Senzing Entity Search WebApp is viewable at
   [http://senzing-entity-search.local](http://senzing-entity-search.local).
1. See
   [additional tips](https://github.com/Senzing/knowledge-base/blob/master/lists/docker-compose-demo-tips.md#senzing-entity-search-webapp)
   for working with Senzing Entity Search WebApp.

#### View Senzing Configurator

:thinking: "Senzing Configuration" is an [optional chart](#install-senzing-configurator-helm-chart).
If the chart has been deployed, it can be viewed.

1. If not done previously, [modify hosts file](#modify-hosts-file)
1. Senzing Configurator is viewable at
   [http://senzing-configurator.local/datasources](http://senzing-configurator.local/datasources).
1. See
   [additional tips](https://github.com/Senzing/knowledge-base/blob/master/lists/docker-compose-demo-tips.md#senzing-configurator)
   for working with Senzing Configurator.

## Cleanup

### Delete everything in project

1. Example:

    ```console
    helm delete --purge ${DEMO_PREFIX}-senzing-debug
    helm delete --purge ${DEMO_PREFIX}-senzing-configurator
    helm delete --purge ${DEMO_PREFIX}-senzing-redoer
    helm delete --purge ${DEMO_PREFIX}-senzing-entity-search-web-app
    helm delete --purge ${DEMO_PREFIX}-senzing-api-server
    helm delete --purge ${DEMO_PREFIX}-senzing-stream-loader
    helm delete --purge ${DEMO_PREFIX}-senzing-init-container
    helm delete --purge ${DEMO_PREFIX}-senzing-mock-data-generator
    helm delete --purge ${DEMO_PREFIX}-rabbitmq
    helm delete --purge ${DEMO_PREFIX}-senzing-ibm-db2
    helm delete --purge ${DEMO_PREFIX}-ibm-db2-driver-installer
    helm delete --purge ${DEMO_PREFIX}-senzing-yum
    helm repo remove senzing
    oc delete -f ${KUBERNETES_DIR}/security-context-constraint-limited.yaml
    oc delete -f ${KUBERNETES_DIR}/security-context-constraint-runasany.yaml
    oc delete -f ${KUBERNETES_DIR}/persistent-volume-claim-senzing.yaml
    oc delete -f ${KUBERNETES_DIR}/persistent-volume-claim-rabbitmq.yaml
    oc delete -f ${KUBERNETES_DIR}/persistent-volume-claim-db2.yaml
    oc delete project ${DEMO_NAMESPACE}
    ```

### Delete minishift cluster

1. Example:

    ```console
    minishift stop
    minishift delete --force --clear-cache
    ```

### Restore hosts file

In the [Modify hosts file](#modify-hosts-file) step,
the `/etc/hosts` file was modified.
Restore contents to the original.

1. Find the original version of the file.
   Example:

    ```console
    $ ls -la /etc | grep hosts

    -rw-r--r--   1 root    root       351 Mar 10 13:57 hosts
    -rw-r--r--   1 root    root       248 Mar 10 13:56 hosts.1583862997
    -rw-r--r--   1 root    root       411 Feb  9  2019 hosts.allow
    -rw-r--r--   1 root    root       711 Feb  9  2019 hosts.deny
    ```

1. :pencil2: Identify the timestamp of the original `/etc/hosts`.
   Example:

    ```console
    export ETC_HOSTS_TIMESTAMP=1583862997
    ```

1. Copy the original version back to `/etc/hosts.
   Example:

    ```console
    sudo cp /etc/hosts.${ETC_HOSTS_TIMESTAMP}  /etc/hosts
    ```

### Delete git repository

1. Delete git repository.  Example:

    ```console
    sudo rm -rf ${GIT_REPOSITORY_DIR}
    ```

## Support

If the instructions don't address an issue you are seeing, please "submit a request" so we can help you.
Here are 3 ways to request:

1. [Report an issue on GitHub](https://github.com/Senzing/ibm-openshift-guide/issues)
1. [Submit a request](https://senzing.zendesk.com/hc/en-us/requests/new)
1. Email: [support@senzing.com](mailto:support@senzing.com)

This repository is a community project.
Feel free to submit a Pull Request for change.

## References

1. [Helm Charts](https://github.com/Senzing/awesome#helm-charts)
1. [Docker images on Docker Hub](https://github.com/Senzing/awesome#dockerhub)
1. [Dockerfiles](https://github.com/Senzing/awesome#dockerfiles)
1. [OKD](https://docs.okd.io/)
    1. [minishift](https://docs.okd.io/latest/minishift/index.html)
        1. [Minishift basic usage](https://docs.okd.io/latest/minishift/using/basic-usage.html)
1. [github.com/minishift/minishift](https://github.com/minishift/minishift/)
    1. [github.com/minishift/minishift-addons/helm](https://github.com/minishift/minishift-addons/tree/master/add-ons/helm)
1. [Adding Persistent Storage to Minishift / CDK 3 in Minutes](https://developers.redhat.com/blog/2017/04/05/adding-persistent-storage-to-minishift-cdk-3-in-minutes/)
