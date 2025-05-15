# Pattern 4: Helm Chart for Distributed API-M Deployment with Traffic Manager Separated from the Control Plane

This is the standard distributed deployment for API Manager. The default configuration consists of two API control planes, two Traffic Managers, and two Universal Gateways. This is a production-ready deployment pattern.

![WSO2 API Manager pattern 4 deployment](https://apim.docs.wso2.com/en/4.5.0/assets/img/setup-and-install/deployment-tm.png)


For advanced details on the deployment pattern, please refer to the official
[documentation](https://apim.docs.wso2.com/en/latest/install-and-setup/setup/single-node/all-in-one-deployment-overview/#single-node-deployment).

## Contents
- [Pattern 4: API-M Deployment with API Control Plane, Traffic Manager and Universal Gateway](#pattern-4-helm-chart-for-distributed-api-m-deployment-with-traffic-manager-separated-from-the-control-plane)
  - [Contents](#contents)
  - [Prerequisites](#prerequisites)
  - [Setup](#setup)
    - [1. Configuring docker images](#1-configuring-docker-images)
      - [1.1. Additional Configurations](#11-additional-configurations)
    - [2. Adding ingress controller](#2-adding-ingress-controller)
  - [Configuration](#configuration)
    - [1. Configuring helm charts](#1-configuring-helm-charts)
      - [1.1 Mounting Keystore and Truststore using a Kubernetes Secret](#11-mounting-keystore-and-truststore-using-a-kubernetes-secret)
      - [1.2 Encrypting secrets](#12-encrypting-secrets)
      - [1.3 Updating the Helm Chart](#13-updating-the-helm-chart)
      - [1.4  Managing Java Keystores and Truststores](#14--managing-java-keystores-and-truststores)
      - [1.5 Configuring SSL in Service Exposure](#15-configuring-ssl-in-service-exposure)
    - [2. Install the Helm Chart](#2-install-the-helm-chart)
    - [3. Add a DNS record mapping the hostnames and the external IP](#3-add-a-dns-record-mapping-the-hostnames-and-the-external-ip)
    - [4. Access Management Consoles](#4-access-management-consoles)
  - [Minimal Configuration](#minimal-configuration)

## Prerequisites

- WSO2 product Docker images used for the Kubernetes deployment.
  
  WSO2 product Docker images available at [DockerHub](https://hub.docker.com/u/wso2/) package General Availability (GA)
  versions of WSO2 products with no [WSO2 Updates](https://wso2.com/updates).

  For a production grade deployment of the desired WSO2 product-version, it is highly recommended to use the relevant
  Docker image which packages WSO2 Updates, available at [WSO2 Private Docker Registry](https://docker.wso2.com/). In order
  to use these images, you need an active [WSO2 Subscription](https://wso2.com/subscription).

- Install [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git), [Helm](https://helm.sh/docs/intro/install/)
  and [Kubernetes client](https://kubernetes.io/docs/tasks/tools/install-kubectl/) in order to run the steps provided in the
  following quick start guide.
- An already setup [Kubernetes cluster](https://kubernetes.io/docs/setup).
- Install [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/). 
- Add the WSO2 Helm chart repository.

    ```
     helm repo add wso2 https://helm.wso2.com && helm repo update
    ```

## Setup
### 1. Configuring docker images
 - WSO2 Product Docker images are required for the deployment. You could either use the images available in the WSO2 private docker registry or you could build your own images.
 - It is recommended to push your own images to the cloud provider's container registry (ACR, ECR, etc.) as a best practice. In order to obtain a docker image for each product please refer to [U2 documentation](https://updates.docs.wso2.com/en/latest/updates/how-to-use-docker-images-to-receive-updates/).
  - You can also use your locally built docker images as well after adding relevant configurations in the values.yaml file under the following section.
  
    ```yaml
    deployment:
      image:
        imagePullSecrets:
          enabled: false
          username: ""
          password: ""
        registry: ""
        repository: ""
        digest: ""
        imagePullPolicy: Always
    ```
  - If you are using a private Docker registry, you must enable `imagePullSecrets.enabled` and provide the username and password.
  - You need [Docker](https://www.docker.com/get-docker) v20.10.x or above to build the custom docker images.
  - Base Dockerfiles can be obtained for APIM using https://github.com/wso2/docker-apim
      >   You need a valid WSO2 subscription to obtain the **U2 updated** docker images from the WSO2 private registry. 


#### 1.1. Additional Configurations
- Since the products need to connect to databases at runtime, we need to include the relevant JDBC drivers in the distribution. This too can be included in the docker image building stage. For example, you can add the MySQL driver as follows.
    ```
    ADD --chown=wso2carbon:wso2 https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.28/mysql-connector-java-8.0.28.jar ${WSO2_SERVER_HOME}/repository/components/lib
    ```
- Furthermore, if there are any customizations to the jars in the product, that too can be included in the docker image itself rather than mounting those from the deployment level (assuming that they are common to all environments).
- Following is a sample Dockerfile to build a custom WSO2 APIM image. Depending on the requirement you may refer to the following and do the necessary additions. The below script will do the following,
Use WSO2 APIM 4.5.0 as the base image
Change UID and GID to 10001. Default APIM image has 802 as UID and GID
Copy 3rd party libraries to the <APIM_HOME>/lib directory
    ```
    FROM docker.wso2.com/wso2am:4.5.0.0

    # Change UID and GID
    USER root
    RUN usermod -u 10001 wso2carbon
    RUN groupmod -g 10001 wso2

    # Switch back to non-root WSO2 user
    USER wso2carbon

    ARG USER_HOME=/home/${USER}
    ARG WSO2_SERVER_NAME=wso2am
    ARG WSO2_SERVER_VERSION=4.5.0
    ARG WSO2_SERVER=${WSO2_SERVER_NAME}-${WSO2_SERVER_VERSION}
    ARG WSO2_SERVER_HOME=${USER_HOME}/${WSO2_SERVER}

    # Copy jdbc mysql driver
    ADD --chown=wso2carbon:wso2 https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.28/mysql-connector-java-8.0.28.jar ${WSO2_SERVER_HOME}/repository/components/lib
    ```

- Once the required changes have been done to the Dockerfile you can use the following command to build the custom image. You will need to replace CONTAINER_REGISTRY, IMAGE_REPO and TAG accordingly.
    ```
    docker build -t CONTAINER_REGISTRY/IMAGE_REPO:TAG .
    ```

### 2. Adding ingress controller

The recommendation is to use [**NGINX Ingress Controller**](https://kubernetes.github.io/ingress-nginx/deploy/) suitable for your cloud environment or local deployment. Some sample annotations that could be used with the ingress resources are as follows.

  - The ingress class should be set to nginx in the ingress resource if you are using the NGINX Ingress Controller.
  - Following are some of the recommended annotations to include in the helm charts for ingresses. These may vary depending on the requirements. Please refer to the [documentation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) for more information about the annotations.
  
    ```yaml
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
      nginx.ingress.kubernetes.io/affinity: "cookie"
      nginx.ingress.kubernetes.io/session-cookie-name: "route"
      nginx.ingress.kubernetes.io/proxy-buffering: "on"
      nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"
    ```
  - You need to create a kubernetes secret including the certificate and the private key and include the name of the secret in the helm charts. This will be used for TLS termination in load balancer level by the ingress controller. Please refer to the [documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls) for more information.
    ```
    kubectl create secret tls my-tls-secret --key <private key filename> --cert <certificate filename>
    ```

### 3. Configuring the database
Before running the API Manager, you must configure the databases and populate them with the initial data. All required database scripts are available in the `dbscripts` directory of the product pack. Locate the appropriate scripts for your chosen database engine and execute them accordingly. It is recommended to use two separate database users with limited permissions for enhanced security.

An example for MySQL is provided below.
```sql
CREATE DATABASE apim_db character set latin1;
CREATE DATABASE shared_db character set latin1;

GRANT ALL ON apim_db.* TO 'apimadmin'@'%';

CREATE USER 'sharedadmin'@'%' IDENTIFIED BY 'sharedadmin';
GRANT ALL ON shared_db.* TO 'sharedadmin'@'%';

```
```bash
mysql -h <DB_HOST> -P 3306 -u sharedadmin -p -Dshared_db < './dbscripts/mysql.sql';
mysql -h <DB_HOST> -P 3306 -u apimadmin -p -Dapim_db < './dbscripts/apimgt/mysql.sql';
```

## Configuration
### 1. General Configuration of Helm Charts

The helm charts for the API Manager deployment are available in the [WSO2 Helm Chart Repository](https://github.com/wso2/helm-apim). You can either use the charts from the repository or clone the repository and use the charts from the local copy.
- The helm naming convention for APIM follows a simple pattern. The following format is used for naming the resources.
```<RELEASE_NAME>-<CHART_NAME>-<RESOURCE_NAME>```

#### 1.1 Mounting Keystore and Truststore using a Kubernetes Secret

- If you are not including the keystore and truststore into the docker image, you can mount them using a Kubernetes secret. Following steps shows how to mount the keystore and truststore using a Kubernetes secret.
- Create a Kubernetes secret with the keystore and truststore files. The secret should contain the primary keystore file, secondary keystore file, internal keystore file, and the truststore file. Note that the secret should be created in the same namespace in which you will be setting up the deployment.
- Make sure to use the same secret name when creating the secret and when configuring the helm chart.
- If you are using a different keystore file name and alias, make sure to update the helm chart configurations accordingly.
In addition to the primary, internal keystores and truststore files, you can also include the keystores for HTTPS transport as well.
- Refer the following sample command to create the secret and use it in the APIM.
  
  ```
  kubectl create secret generic jks-secret --from-file=wso2carbon.jks --from-file=client-truststore.jks --from-file=wso2internal.jks -n <namespace>
  ```

#### 1.2 Encrypting secrets

- If you need to use cipher tool to encrypt the passwords in the secret, first you need to encrypt the passwords using the cipher tool. The cipher tool can be found in the bin directory of the product pack. The following command can be used to encrypt the password.
  ```
  sh cipher-tool.sh -Dconfigure
  ```
- Also the apictl can be used to encrpypt password as well. Reference can be found in [following](https://apim.docs.wso2.com/en/latest/install-and-setup/setup/api-controller/encrypting-secrets-with-ctl/).
- Then the encrypted values should be filled in the the relevant fields of values.yaml.
- Since internal keystore password is required to resolve the encrypted value in runtime, we need to store the value in the cloud provider's secret manager. You can use the cloud provider's secret store to store the password of the internal keystore. The following section can be used to add the cloud provider's credentials to fetch the internal keystore password. Configuration for aws can be at as below. 
  ```yaml
  internalKeystorePassword:
    # -- AWS Secrets Manager secret name
    secretName: ""
    # -- AWS Secrets Manager secret key
    secretKey: ""
  ```
  > Please note that currently  AWS, Azure and GCP Secrets Managers are only supported for this.



#### 1.3 Updating the Helm Chart

 - Add the following configurations to reflect the docker image created previously in the helm chart.
  
    ```yaml
    wso2:
      deployment:
        image:
          imagePullSecrets:
            enabled: false
            username: ""
            password: ""		
          registry: ""
          repository: ""
          digest: ""
    ```
 - Provide the database configurations under the following section.

    ```yaml
    wso2:
      apim:
        configurations:
          databases:
            apim_db:
              url: ""
              username: ""
              password: ""
            shared_db:
              url: ""
              username: ""
              password: ""
    ```
    - If you need to change the hostnames, update them under the Kubernetes ingress section.
    - Update the admin credentials in the configuration directory.
    - Update the keystore passwords in the security section of the `values.yaml` file.
    - Review the descriptions of other configurations and modify them as needed to meet your requirements. A simple deployment can be achieved using the basic configurations provided in the `values.yaml` file. All configurations for this Helm chart are documented in the [official documentation](https://github.com/wso2/helm-apim/blob/main/all-in-one/README.md).
    - Update the admin username and password as required.
    ```yaml
      # -- Super admin username
      adminUsername: ""
      # -- Super admin password
      adminPassword: ""
    ```

#### 1.4  Managing Java Keystores and Truststores

* By default, this deployment uses the default keystores and truststores provided by the relevant WSO2 product.

* For advanced details with regards to managing custom Java keystores and truststores in a container based WSO2 product deployment
  please refer to the [official WSO2 container guide](https://github.com/wso2/container-guide/blob/master/deploy/Managing_Keystores_And_Truststores.md).
  
#### 1.5 Configuring SSL in Service Exposure

* For WSO2 recommended best practices in configuring SSL when exposing the internal product services to outside of the Kubernetes cluster,
  please refer to the [official WSO2 container guide](https://github.com/wso2/container-guide/blob/master/route/Routing.md#configuring-ssl).


### 2. API Control Plane Configurations

#### 2.1 Configure multiple gateways

If you need to distribute the Gateway load that comes in, you can configure multiple API Gateway environments in WSO2 API Manager to publish to a single Developer Portal. [See more...](https://apim.docs.wso2.com/en/latest/manage-apis/deploy-and-publish/deploy-on-gateway/deploy-api/deploy-through-multiple-api-gateways/)
```yaml
    gateway:
        # -- APIM Gateway environments
        environments:
        - name: "Default"
          type: "hybrid"
          gatewayType: "Regular"
          provider: "wso2"
          visibility:
          displayInApiConsole: true
          description: "This is a hybrid gateway that handles both production and sandbox token traffic."
          showAsTokenEndpointUrl: true
          serviceName: "apim-gw-wso2am-gateway-service"
          servicePort: 9443
          wsHostname: "websocket.wso2.com"
          httpHostname: "gw.wso2.com"
          websubHostname: "websub.wso2.com"
        - name: "Default_apk"
          type: "hybrid"
          provider: "wso2"
          gatewayType: "APK"
          displayInApiConsole: true
          description: "This is a hybrid gateway that handles both production and sandbox token traffic."
          showAsTokenEndpointUrl: true
          serviceName: "apim-gw-wso2am-gateway-service"
          servicePort: 9443
          wsHostname: "websocket.wso2.com"
          httpHostname: "default.gw.wso2.com:9095"
          websubHostname: "websub.wso2.com"

```

#### 2.2 Configure User Store Properties

You can configure user store properties as described in this [documentation](https://apim.docs.wso2.com/en/latest/administer/managing-users-and-roles/managing-user-stores/working-with-properties-of-user-stores/):

```yaml
    userStore:
    # -- User store type.
    type: "database_unique_id"
    # -- User store properties
    properties:
        ReadGroups: true
```

> **Important:** If you do not want to configure any of the above properties, you must remove the `properties` block from the YAML file.

#### 2.4 Configure JWKS URL
By default, for the super tenant, the Resident Key Manager's JWKS URL is set to `https://localhost:9443/oauth2/jwks`. You can configure this URL for the super tenant using the Helm chart as shown below:

```yaml
wso2:
  apim:
    configurations:
      oauth_config:
        oauth2JWKSUrl: "https://<ACP_SERVICE_NAME>:9443/oauth2/jwks"
```
#### 2.5 Deploy ACP

Now deploy the Helm Chart using the following command after creating a namespace for the deployment. Replace <release-name> and <namespace> with appropriate values. Replace <helm-chart-path> with the path to the Helm Deployment.
  
  ```bash
  kubectl create namespace <namespace>
  helm install <release-name> <helm-chart-path> --version 4.5.0-1 --namespace <namespace> --dependency-update --create-namespace
  ```


### 3. Traffic Manager Configurations

#### 3.1 Configure Key Manager and Eventhub

- In this pattern, the ACP is used as the Key Manager. Therefore, you need to specify the ACP service URL as the Key Manager service URL.
  ```yaml
    km:
      # -- Key manager service name if default Resident KM is used
      serviceUrl: "<ACP_SERVICE_NAME>"
  ```
- Configure eventhub
  ```yaml
  eventhub:
    # -- Event hub (control plane) loadbalancer service url
    serviceUrl: "<ACP_SERVICE_NAME>"
    # -- Event hub service urls
    urls:
      - "<ACP-1_SERVICE_NAME>"
      - "<ACP-2_SERVICE_NAME>"
  ```

#### 3.2 Deploy TM

Replace <release-name> and <namespace> with appropriate values. Replace <helm-chart-path> with the path to the Helm Deployment.
  
  ```bash
  helm install <release-name> <helm-chart-path> --version 4.5.0-1 --namespace <namespace> --dependency-update --create-namespace
  ```

### 4. Universal Gateway Configuration

#### 4.1 Configure Key Manager, Eventhub and Throttling
- Configure ACP as the Key Manager
  ```yaml
    km:
      # -- Key manager service name if default Resident KM is used
      serviceUrl: "<ACP_SERVICE_NAME>"
  ```
- Configure eventhub
  ```yaml
  eventhub:
    # -- Event hub (control plane) loadbalancer service url
    serviceUrl: "<ACP_SERVICE_NAME>"
    # -- Event hub service urls
    urls:
      - "<ACP-1_SERVICE_NAME>"
      - "<ACP-2_SERVICE_NAME>"
  ```
- Configure throttling
  ```yaml
  throttling:
    serviceUrl: "<TM_SERVICE_NAME>"
    portOffset: 0
    # -- Port of the service URL
    servicePort: 9443
    # -- Traffic manager service urls. You only need to define one if the TM is not in HA.
    urls:
      - "<TM-1_SERVICE_NAME>"
      - "<TM-2_SERVICE_NAME>"
    # -- Enable unlimited throttling tier
    unlimitedTier: true
    # -- Enable header based throttling
    headerBasedThrottling: false
    # -- Enable JWT claim based throttling
    jwtClaimBasedThrottling: false
    # -- Enable query param based throttling
    queryParamBasedThrottling: false
  ```

#### 4.2 Deploy Universal Gateway

Replace <release-name> and <namespace> with appropriate values. Replace <helm-chart-path> with the path to the Helm Deployment.
  
  ```bash
  helm install <release-name> <helm-chart-path> --version 4.5.0-1 --namespace <namespace> --dependency-update --create-namespace
  ```

### 5. Add a DNS record mapping the hostnames and the external IP

Obtain the external IP (EXTERNAL-IP) of the API Manager Ingress resources, by listing down the Kubernetes Ingresses.
```
kubectl get ing -n <NAMESPACE>
```

If the defined hostnames (in the previous step) are backed by a DNS service, add a DNS record mapping the hostnames and
the external IP (`EXTERNAL-IP`) in the relevant DNS service.

If the defined hostnames are not backed by a DNS service, for the purpose of evaluation you may add an entry mapping the
hostnames and the external IP in the `/etc/hosts` file at the client-side.

```
<EXTERNAL-IP> <kubernetes.ingress.management.hostname> <kubernetes.ingress.gateway.hostname> <kubernetes.ingress.websub.hostname> <kubernetes.ingress.websocket.hostname> 
```

### 6. Access Management Consoles

- API Manager Publisher: `https://<kubernetes.ingress.management.hostname>/publisher`

- API Manager DevPortal: `https://<kubernetes.ingress.management.hostname>/devportal`

- API Manager Carbon Console: `https://<kubernetes.ingress.management.hostname>/carbon`

- Universal Gateway: `https://<kubernetes.ingress.gateway.hostname>`

## Minimal Configuration

- We have provided a pre-configured YAML files to help you quickly start the deployment. You can use this file as a starting point to deploy this pattern. This deployment requires separate databases. Therefore, follow the steps in [1.1. Additional Configurations](#11-additional-configurations) to build the Docker images with JDBC drivers, and refer to [3. Configuring the database](#3-configuring-the-database) to set up the database.
- Follow the steps in [1.1 Mounting Keystore and Truststore using a Kubernetes Secret](#11-mounting-keystore-and-truststore-using-a-kubernetes-secret) to create the truststore and keystore. If you want to use the WSO2 default keystore and truststore, you can find them in the `repository/resources/security` directory of the product pack. Navigate to this location and run the following command to create the secret:
```bash
kubectl create secret generic jks-secret --from-file=wso2carbon.jks --from-file=client-truststore.jks
```
- Run the following command to deploy the Helm charts:
> **Important:** Naming conventions are important. If you want to change them, ensure consistency. 

1. Deploy ACP
```bash
helm install apim-acp wso2/wso2-acp -f default_acp_values.yaml
```

2. Deploy TM
```bash
helm install apim-tm wso2/wso2-tm -f default_tm_values.yaml
```

3. Deploy GW
```bash
helm install apim-gw wso2/wso2-tm -f default_gw_values.yaml
```

- Once the service is up and running, deploy the NGINX Ingress Controller by following the steps outlined in [2. Adding ingress controller](#2-adding-ingress-controller).
