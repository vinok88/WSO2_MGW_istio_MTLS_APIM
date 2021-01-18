# API Management for Istio Services

In this scenario, we have several microservices which are deployed in Istio. For applying API management for those microservices, we can expose an API for those microservices. 

This works in Istio permissive mode and Strict MTLS mode.

![Alt text](mtls-mode.png?raw=true "Istio in Permissive mode and MTLS mode")

### Installation Prerequisites

- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

- [Kubernetes v1.15 or above](https://Kubernetes.io/docs/setup/) <br>

    - Minimum CPU : 8vCPU
    - Minimum Memory : 8GB
    
- Download Istio 1.8.1

    ``` sh 
    curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.8.1 sh -
    ```
- Deploy Istio in K8s
    ``` sh
    cd istio-1.8.1/bin
    ./istioctl install --set profile=demo -y
    ```

- An account in DockerHub or private docker registry

- Download [k8s-api-operator-1.2.2.zip](https://github.com/wso2/k8s-api-operator/releases/download/v1.2.2/k8s-api-operator-1.2.2.zip) and extract the zip

    1. This zip contains the artifacts that required to deploy in Kubernetes.
    2. Extract k8s-api-operator-1.2.2.zip
    
    ```sh
    cd k8s-api-operator-1.2.2/scenarios/scenario-13/S02-APIM_for_Istio_Services_MTLS
    ```
 
    **_Note:_** You need to run all commands from within the ```S02-APIM_for_Istio_Services_MTLS``` directory.

<br />

#### Step 1: Configure API Controller

- Download API controller v3.2.0 or the latest v3.2.x from the [API Manager Tooling web site](https://wso2.com/api-management/tooling/)

    - Under Dev-Ops Tooling section, you can download the tool based on your operating system.

- Extract the API controller distribution and navigate inside the extracted folder using the command-line tool

- Add the location of the extracted folder to your system's $PATH variable to be able to access the executable from anywhere.

- You can find available operations using the below command.
    
  ```sh
  >> apictl --help
  ```
<br />

#### Step 2: Install API Operator

- Set the operator version as `v1.2.2` by executing following in a terminal.
    ```sh
    >> export WSO2_API_OPERATOR_VERSION=v1.2.2
    ```
- Execute the following command to install API Operator interactively and configure repository to push the microgateway image.
- Select "Docker Hub" as the repository type.
- Enter repository name of your Docker Hub account (usually it is the username as well).
- Enter username and the password
- Confirm configuration are correct with entering "Y"

    ```sh
    >> apictl install api-operator
    Choose registry type:
    1: Docker Hub
    2: Amazon ECR
    3: GCR
    4: HTTP Private Registry
    5: HTTPS Private Registry
    6: Quay.io
    Choose a number: 1: 1
    Enter repository name: jennifer
    Enter username: jennifer
    Enter password: *******
    
    Repository: jennifer
    Username  : jennifer
    Confirm configurations: Y: Y
    ```

    Output:
    ```
    customresourcedefinition.apiextensions.k8s.io/apis.wso2.com created
    customresourcedefinition.apiextensions.k8s.io/ratelimitings.wso2.com created
    ...
    
    namespace/wso2-system created
    deployment.apps/api-operator created
    ...
    
    [Setting to K8s Mode]
    ```
<br />

#### Step 3: Deploy Microservices

- When you execute this command, it creates a namespace called `micro` and **enable Istio sidecar injection** for that
namespace. Also this deploys 3 microservices.

    ```sh
    >> apictl create -f microservices.yaml
    
    >> apictl get pods -n micro
  
    Output:
    NAME                         READY   STATUS    RESTARTS   AGE
    inventory-7dc5dfdc58-gnxqx   2/2     Running   0          9m
    products-8d478dd48-2kgdk     2/2     Running   0          9m
    review-677dd8fbd8-9ntth      2/2     Running   0          9m
    ```
<br />

#### Step 4: Deploy an API for the microservices

- We are creating a namespace called `wso2` and deploy our API there. In this namespace, we have
**NOT enabled Istio sidecar injection**.

    **Note:** For this sample, using the flag `--override` to update configs, if there are images in the docker registry
    which where created during older versions of API Operator.
   
    ```sh
    >> apictl create ns wso2
    >> apictl add api \
                -n online-store-api-mlts \
                --from-file=./swagger.yaml \
                --namespace=wso2 \
                --override
    
    >> apictl get pods -n wso2
  
    Output:
    NAME                                                             READY   STATUS      RESTARTS   AGE
    online-store-api-mlts-5748695f7b-jxnpf                           1/1     Running     0          14m
    online-store-api-mlts-kaniko-b5hqb                               0/1     Completed   0          14m
    ```
<br />

#### Step 5: Setup routing in Istio

- Due to Strict MTLS in Istio, we are deploying a gateway and a virtual service in Istio.

    ```sh
    >> apictl create -f gateway-virtualservice.yaml
    ```
<br />

#### Step 6: Invoke the API
 
- Retrieve the API service endpoint details
 
     The API service is exposed as the Load Balancer service type. You can get the API service endpoint details by using the following command.
 
     ```sh
     >> apictl get services -n wso2
     
     Output:
     NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                         AGE
     online-store-api-mlts  LoadBalancer   10.83.9.142    35.232.188.134   9095:31055/TCP,9090:32718/TCP   57s
     ```
 
 <details><summary>If you are using Minikube click here</summary>
 <p>
 
 **_Note:_**  By default API operator requires the LoadBalancer service type which is not supported in Minikube by default. Here is how you can enable it on Minikube.
 
 - On Minikube, the LoadBalancer type makes the Service accessible through the minikube service command.
 
     ```sh
     >> minikube service <SERVICE_NAME> --url
     >> minikube service online-store --url
     ```
     
     The IP you receive from above output can be used as the "external-IP" in the following command.
 
 </p>
 </details>
 
---
 
 - Invoke the API as a regular microservice
 
     Let’s observe what happens if you try to invoke the API as a regular microservice.
     ```sh
     >> curl -X GET "https://<EXTERNAL-IP>:9095/storemep/v1.0.0/products" -k
     ```
     
     You will get an error as below.
     
     ```json
     {
         "fault": {
             "code": 900902,
             "message": "Missing Credentials",
             "description": "Missing Credentials. Make sure your API invocation call has a header: \"Authorization\""
         }
     }
     ```
     
     Since the API is secured now, you are experiencing the above error. Hence you need a valid access token to invoke the API.
     
 - Invoke the API with an access token
 
     You can find a sample token below.
     
     ```sh
    TOKEN=eyJ4NXQiOiJNell4TW1Ga09HWXdNV0kwWldObU5EY3hOR1l3WW1NNFpUQTNNV0kyTkRBelpHUXpOR00wWkdSbE5qSmtPREZrWkRSaU9URmtNV0ZoTXpVMlpHVmxOZyIsImtpZCI6Ik16WXhNbUZrT0dZd01XSTBaV05tTkRjeE5HWXdZbU00WlRBM01XSTJOREF6WkdRek5HTTBaR1JsTmpKa09ERmtaRFJpT1RGa01XRmhNelUyWkdWbE5nX1JTMjU2IiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJhZG1pbkBjYXJib24uc3VwZXIiLCJhdWQiOiJKRmZuY0djbzRodGNYX0xkOEdIVzBBR1V1ME1hIiwibmJmIjoxNTk3MjExOTUzLCJhenAiOiJKRmZuY0djbzRodGNYX0xkOEdIVzBBR1V1ME1hIiwic2NvcGUiOiJhbV9hcHBsaWNhdGlvbl9zY29wZSBkZWZhdWx0IiwiaXNzIjoiaHR0cHM6XC9cL3dzbzJhcGltOjMyMDAxXC9vYXV0aDJcL3Rva2VuIiwiZXhwIjoxOTMwNTQ1Mjg2LCJpYXQiOjE1OTcyMTE5NTMsImp0aSI6IjMwNmI5NzAwLWYxZjctNDFkOC1hMTg2LTIwOGIxNmY4NjZiNiJ9.UIx-l_ocQmkmmP6y9hZiwd1Je4M3TH9B8cIFFNuWGHkajLTRdV3Rjrw9J_DqKcQhQUPZ4DukME41WgjDe5L6veo6Bj4dolJkrf2Xx_jHXUO_R4dRX-K39rtk5xgdz2kmAG118-A-tcjLk7uVOtaDKPWnX7VPVu1MUlk-Ssd-RomSwEdm_yKZ8z0Yc2VuhZa0efU0otMsNrk5L0qg8XFwkXXcLnImzc0nRXimmzf0ybAuf1GLJZyou3UUTHdTNVAIKZEFGMxw3elBkGcyRswzBRxm1BrIaU9Z8wzeEv4QZKrC5NpOpoNJPWx9IgmKdK2b3kIWJEFreT3qyoGSBrM49Q
     ```
     Copy and paste the above token in the command line. Now you can invoke the API using the cURL command as below.
     
     ```sh
     Format: 
     
     >> curl -X GET "https://<EXTERNAL-IP>:9095/<API-context>/<API-resource>" -H "Authorization:Bearer $TOKEN" -k
     ```
 
     Example commands:
     
     ```sh
     >> curl -X GET "https://35.232.188.134:9095/storemep/v1.0.0/products" -H "Authorization:Bearer $TOKEN" -k
     
     >> curl -X GET "https://35.232.188.134:9095/storemep/v1.0.0/products/101" -H "Authorization:Bearer $TOKEN" -k
          
     >> curl -X GET "https://35.232.188.134:9095/storemep/v1.0.0/review/101" -H "Authorization:Bearer $TOKEN" -k
     
     >> curl -X GET "https://35.232.188.134:9095/storemep/v1.0.0/inventory/101" -H "Authorization:Bearer $TOKEN" -k
     ```

 <br />


Optionally follow the below steps to install a API manager instance and security token service so that you can generate security tokens on your own.

#### Install the API portal and security token service

[WSO2AM Kubernetes Operator](https://github.com/wso2/k8s-wso2am-operator) is used to deploy API portal and security
token service.

- Install the WSO2AM Operator in Kubernetes.

    ```sh
    >> apictl install wso2am-operator
    
    namespace/wso2-system created
    serviceaccount/wso2am-pattern-1-svc-account created
    ...
    configmap/wso2am-p1-apim-2-conf created
    configmap/wso2am-p1-mysql-dbscripts created
    [Setting to K8s Mode]
    ```

- Install API Portal and security token service under a namespace called "wso2"
    ```sh
    >> apictl apply -f k8s-artifacts/wso2am-operator/api-portal/
    
    Output:
    namespace/wso2 created
    configmap/apim-conf created
    apimanager.apim.wso2.com/custom-pattern-1 created
    ```

- Access API Portal and security token service

    ```sh
    >> apictl get pods -n wso2
    
    Output:
    NAME                        READY   STATUS    RESTARTS   AGE
    wso2apim-596f9d5ff6-mcch6   1/1     Running   0          5m7s
        
    >> apictl get services -n wso2
    
    Output:
    NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                       AGE
    wso2apim   NodePort   10.96.209.21   <none>        8280:32004/TCP,8243:32003/TCP,9763:32002/TCP,9443:32001/TCP   5m24s
    ```

    **_Note:_** To access the API portal, add host mapping entry to the /etc/hosts file. As we have exposed
    the API portal service in Node Port type, you can use the IP address of any Kubernetes node.
    
    ```
    <ANY_K8S_NODE_IP>  wso2apim
    ```
    
    - For Docker for Mac use "127.0.0.1" for the K8s node IP
    - For Minikube, use minikube ip command to get the K8s node IP
    - For GKE
        ```$xslt
        (apictl get nodes -o jsonpath='{ $.items[*].status.addresses[?(@.type=="ExternalIP")].address }')
        ```
        - This will give the external IPs of the nodes available in the cluster. Pick any IP to include in /etc/hosts file.
      
       **API Portal** - https://wso2apim:32001/devportal 

<br />

Pushing the API to the API Portal


To make the API discoverable for other users and get the access tokens, we need to push the API to the API portal.
Then the app developers/subscribers can navigate to the devportal (https://wso2apim:32001/devportal)
to perform the following actions.

- Create an application
- Subscribe the API to the application
- Generate a JWT access token 

The following commands will help you to push the API to the API portal in Kubernetes.
Commands of the API Controller can be found [here](https://github.com/wso2/product-apim-tooling/blob/3.2.x/import-export-cli/docs/apictl.md) 

- Go to scenario directory
    ```sh
    >> cd ./S02-APIM_for_Istio_Services_MTLS
- Add the API portal as an environment to the API controller using the following command.

    ```sh
    >> apictl add-env -e k8s \
                --apim https://wso2apim:32001 \
                --token https://wso2apim:32001/oauth2/token
    
    Output:
    Successfully added environment 'k8s'
    ```

- Initialize the API project using API Controller

    ```sh
    >> apictl init online-store \
                --oas=./swagger.yaml \
                --initial-state=PUBLISHED
    
    Output:
    Initializing a new WSO2 API Manager project in /home/wso2/k8s-api-operator/scenarios/scenario-1/online-store
    Project initialized
    Open README file to learn more
    ```

- Import the API to the API portal. **[IMPORTANT]**

    For testing purpose use ***admin*** as username and password when prompted.
    </br>
    
    ```sh
    >> apictl login k8s -k
    >> apictl import-api -f online-store/ -e k8s -k
    
    Output:
    Successfully imported API
    ```
<br />

#### Step 8: Generate an access token for the API

- By default the API is secured with JWT. Hence a valid JWT token is needed to invoke the API.
You can obtain a JWT token using the API Controller command as below.

    ```sh
    >> apictl get-keys -n online-store-api -v v1.0.0 -e k8s --provider admin -k
    
    Output:
    API name:  OnlineStore & version:  v1.0.0 exists
    API  OnlineStore : v1.0.0 subscribed successfully.
    Access Token:  eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6IlpqUm1ZVE13TlRKak9XVTVNbUl6TWpnek5ESTNZMkl5TW1JeVkyRXpNamRoWmpWaU1qYzBaZz09In0.eyJhdWQiOiJodHRwOlwvXC9vcmcud3NvMi5hcGltZ3RcL2dhdGV3YXkiLCJzdWIiOiJhZG1pbkBjYXJib24uc3VwZXIiLCJhcHBsaWNhdGlvbiI6eyJvd25lciI6ImFkbWluIiwidGllciI6IlVubGltaXRlZCIsIm5hbWUiOiJkZWZhdWx0LWFwaWN0bC1hcHAiLCJpZCI6MiwidXVpZCI6bnVsbH0sInNjb3BlIjoiYW1fYXBwbGljYXRpb25fc2NvcGUgZGVmYXVsdCIsImlzcyI6Imh0dHBzOlwvXC93c28yYXBpbTozMjAwMVwvb2F1dGgyXC90b2tlbiIsInRpZXJJbmZvIjp7IlVubGltaXRlZCI6eyJzdG9wT25RdW90YVJlYWNoIjp0cnVlLCJzcGlrZUFycmVzdExpbWl0IjowLCJzcGlrZUFycmVzdFVuaXQiOm51bGx9fSwia2V5dHlwZSI6IlBST0RVQ1RJT04iLCJzdWJzY3JpYmVkQVBJcyI6W3sic3Vic2NyaWJlclRlbmFudERvbWFpbiI6ImNhcmJvbi5zdXBlciIsIm5hbWUiOiJPbmxpbmUtU3RvcmUiLCJjb250ZXh0IjoiXC9zdG9yZVwvdjEuMC4wXC92MS4wLjAiLCJwdWJsaXNoZXIiOiJhZG1pbiIsInZlcnNpb24iOiJ2MS4wLjAiLCJzdWJzY3JpcHRpb25UaWVyIjoiVW5saW1pdGVkIn1dLCJjb25zdW1lcktleSI6Im1Hd0lmUWZuZHdZTVZxT25JVW9Rczhqc1B0Y2EiLCJleHAiOjE1NzIyNjAyMjQsImlhdCI6MTU3MjI1NjYyNCwianRpIjoiNTNlYWJkYWEtY2IyZC00MTQ0LWEzYWUtZDNjNTIxMjgwYjM4In0.QU9rt4WBLcIOXzDkdiBpo_SAN_W4jpMlymPSgdhe4mf4FmdepA6hIXa_NXdzWyOST2XcHskWleL-9bhv4GecvDaCcMUwfSKOo_8DuphYhtv0BukpGpyfzK2SZDtABxxtdRUmNDcyXJiC5NU4laXlDGzUruI_LISjkeeCaK4gA93YQC3Nd0xe14uIO940UNsSiUuI5cZkeKlB9k5vKIzjN1-M-SJCvtDkusvdPTgkSHZL29ICsMQl9rTSRm6dL4xq9rcH7osD-o_amgurkm1RvNagzN0buku6y4tuEyisZvRUlNkQ2KRzX6E6VwNKHAFQ7CG95-k-QYvXDGDXYGNisw  
    ```
  
    **_Note:_** You also have the option to generate a token by logging into the devportal. 

<br />

### Documentation

You can find the documentation [here](docs/Readme.md).


### Cleanup

Execute the following commands if you wish to clean up the Kubernetes cluster by removing all the applied artifacts
and configurations related to API operator and API portal.

```sh
>> apictl remove env k8s;
   apictl set --mode k8s;
   apictl delete api online-store;
   apictl delete -f scenarios/scenario-1/products_dep.yaml;
   apictl delete -f k8s-artifacts/api-portal/;
   apictl uninstall api-operator;
```
