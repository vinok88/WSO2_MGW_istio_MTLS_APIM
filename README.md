This demonstrates the installation of WSO2 APIM Microgateway to manage APIs in a Istio environment. 

Follow the steps in [S02-APIM_for_Istio_Services_MTLS](https://github.com/vinok88/WSO2_MGW_istio_MTLS_APIM/tree/master/S02-APIM_for_Istio_Services_MTLS) scenario.

Optionally follow the below steps to install a API manager instance and security token service.

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
