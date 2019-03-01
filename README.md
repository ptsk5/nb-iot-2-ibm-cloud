# Integration of NB-IoT device with IBM Cloud
## About
Following steps describe an example of integration between the device, which communicates over the Narrowband IoT (NB-IoT) Low Power Wide Area Network (LPWAN) provided by a local operator, and IBM Cloud. 

![Overview](/doc_images/overview.png)

Narrowband IoT device will be simulated by [SerialDeviceApp](https://github.com/ptsk5/serial-device-app) that will communicate with [Quectel GSM/NB-IoT EVB Kit](https://www.quectel.com/product/gsmevb.htm) fitted with [BC95-B20](https://www.quectel.com/product/bc95.htm) high-performance NB-IoT module. The communication will be realised via Serial port over USB by using AT commands.


## IBM Cloud components

Nowadays, IBM Cloud offers many integration options on several levels, e.g. bare metal servers, virtual machines, containers, Platform as a Service (PaaS) components,... In the context of this example we will use: 

* **Kubernetes Service** - to create a private Kubernetes (K8s) cluster and use it as an entry point for data coming from the NB-IoT device.
* **Container Registry** - private registry where we can push and manage Docker container images, and then run them in IBM Cloud Kubernetes Service.
* As part of public **Cloud Foundry** environment, we will provision **Internet of Things Platform Starter** package. This starter includes: 
    - **Internet of Things Platform** service,
    - **Cloudant** database, 
    - **SDK for Node.js** which comes with pre-installed Node-Red application.

## Assumptions and limitations
The reader is expected to have at least a free account at IBM Cloud (you can create one [here](https://console.bluemix.net/)). To deploy any Cloud Foundry Application, one has to have a Cloud Foundry space.  


## Data flow

1. The NB-IoT device creates datagram and sends UDP packets to a predefined IP address (public IP address of the worker node of the K8s cluster with a port number mapped to a particular container). We will use node-red container.
2. The container (node-red application) transforms UDP datagram to an MQTT message and sends data to the Internet of Things Platform as a (registered) proxy device. 
3. Node-Red application (_the second completely different instance than the one deployed in the K8s cluster_) deployed in the Cloud Foundry environment implements a gateway which listens to the data of the proxy device, transforms data content, stores data in the Cloudant database, and sends them on behalf of the end device (thus provides auto-registration function).
4. The Node-Red application further provides GUI (node-red-dashboard) for an end user and can send some UDP datagrams back to the NB-IoT device.

![Data Flow](/doc_images/data_flow.png)


## Container Registry

1. Log in to IBM Cloud, find **Container Registry** in the Catalog, and create an instance. 
2. Follow Registry Quick Start and install IBM Cloud CLI, Docker CLI and Container Registry plug-in (if not already installed). 
3. Log in to your IBM Cloud account from CLI, e.g. (depends on your location - Frankfurt, London, Dallas,...):  
    ```
    $ ibmcloud login -a https://api.eu-de.bluemix.net
    ```
4. Create a namespace (e.g. _nbiot_namespace_, in my case):
    ```
    $ ibmcloud cr namespace-add nbiot_namespace
    ```
5. Pull node-red-docker image from Docker Hub:
    ```
    $ docker pull nodered/node-red-docker:latest
    ```
6. Choose a repository (_node-red-docker_, in my case) and tag (I simply used number _1_) by which you can identify the image (look for registry URL in the Quick Start, e.g. _de.icr.io_):
    ```
    $ docker tag nodered/node-red-docker de.icr.io/nbiot_namespace/node-red-docker:1
    ```
7. Push the image into the private registry and verify that your image is in there: 
    ```
    $ docker push de.icr.io/nbiot_namespace/node-red-docker:1
    $ ibmcloud cr image-list
    ```

## Kubernetes Service

### Create a cluster
1. In the IBM Cloud Catalog find **Kubernetes Service**. Create an instance by choosing _Free_ cluster type and by providing _Cluster name_ (e.g. myCluster).
2. Follow Gain access to your cluster to install CLI tools and the Kubernetes Service plug-in (if not already installed).
3. Target the Kubernetes Service region (e.g. _eu-central_), download the Kubernetes configuration files, and export KUBECONFIG environment variable:
    ```
    $ ibmcloud ks region-set eu-central
    $ ibmcloud ks cluster-config myCluster
    $ export KUBECONFIG=/Users/$USER/.bluemix/plugins/container-service/clusters/myCluster/kube-config-mil01-myCluster.yml
    ```
4. Verify that you can connect to the cluster:
    ```
    $ kubectl get nodes
    NAME            STATUS    ROLES     AGE       VERSION
    10.xxx.xxx.xxx   Ready     <none>    8d        v1.10.12+IKS
    ```

### Create a Kubernetes secret to access Container Registry
5. To be able to access IBM Cloud Container Registry from Kubernetes cluster, we need to create a token to grant access to all our namespaces in a region. Create a token by running: 

    ```
    $ ibmcloud cr token-add --description "<This is my token>" --non-expiring -q
    <<long registry token>>
    ```
6. Create new secret to be able to download image from IBM Cloud Container Registry during deployment (`--namespace` parameter can be found by running `kubectl get namespaces` command; `--docker-email` is mandatory to create a Kubernetes secret, but it is not used after creation; `my-image-pull-secret` is my name for this secret, use whatever name you prefer):

    ```
    $ kubectl --namespace default create secret docker-registry my-image-pull-secret  --docker-server=de.icr.io --docker-username=token --docker-password=<<long registry token from the previous step>> --docker-email=fictional@email.address
    ```

### Create a deployment .yaml

7. Let's define a deployment definition (see `mydeployment.yaml`) to be able to deploy a container into a K8s cluster,  Beware of the `image:` to address your image within your namespace and `imagePullSecrets:` in case you provided own naming. 

    ```YAML
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: node-red-deployment
    labels:
        app: node-red
    spec:
    selector:
        matchLabels:
        app: node-red
    template:
        metadata:
        labels:
            app: node-red
        spec:
        containers:
            - name: node-red-docker-container
            image: de.icr.io/nbiot_namespace/node-red-docker:1
            ports:
            - name: httpport
                containerPort: 1880
                protocol: TCP
            - name: udpport
                containerPort: 1882
                protocol: UDP
        imagePullSecrets:
            - name: my-image-pull-secret
    ```
8. Apply `.yaml` definition into default namespace by running: 
    ```
    $ kubectl apply -f mydeployment.yaml -n default 
    ```

### Expose services
9. We need to expose two ports to have access inside the conteiner. One port is used for `http` communication (to access Node-Red flows editor) and the second one is used for `UDP` datagrams. See definitions of two container ports above. 
10. Expose service for http communication:

    ```
    $ kubectl expose deployment/node-red-deployment --type=NodePort --port=1880 --name=node-red-http-service --target-port=httpport
    ```
11. Expose service for udp communication:
    ```
    $ kubectl expose deployment/node-red-deployment --type=NodePort --port=1882 --name=node-red-udp-service --target-port=udpport
    ```
12. Find the NodePorts that are randomly assigned when they are generated with the expose command by running the following command:

    ```
    $ kubectl describe svc 
    ...
    Name:                     node-red-http-service
    Namespace:                default
    Labels:                   app=node-red
    Annotations:              <none>
    Selector:                 app=node-red
    Type:                     NodePort
    IP:                       172.xxx.xxx.xxx
    Port:                     <unset>  1880/TCP
    TargetPort:               httpport/TCP
    NodePort:                 <unset>  30394/TCP
    Endpoints:                172.xxx.xxx.xxx:1880
    Session Affinity:         None
    External Traffic Policy:  Cluster
    Events:                   <none>

    Name:                     node-red-udp-service
    Namespace:                default
    Labels:                   app=node-red
    Annotations:              <none>
    Selector:                 app=node-red
    Type:                     NodePort
    IP:                       172.xxx.xxx.xxx
    Port:                     <unset>  1882/UDP
    TargetPort:               udpport/UDP
    NodePort:                 <unset>  30662/UDP
    Endpoints:                172.xxx.xxx.xxx:1882
    Session Affinity:         None
    External Traffic Policy:  Cluster
    Events:                   <none>
    ```
13. NodePorts as can be see in the previous example are `30394/TCP` for http communication and `30662/UDP` for udp communication.

14. Get the public IP address for the worker node in the cluster.
    ```
    $ ibmcloud ks workers --cluster myCluster
    OK
    ID                                                 Public IP         Private IP      Machine Type   State    Status   Zone    Version   
    kube-mil01-pa66a64ed30035485190c996520469de15-w1   159.xxx.xxx.xxx   10.xxx.xxx.xxx   free           normal   Ready    mil01   1.10.12_1544 
    ```

15. K8s cluster now looks like follows:
    ![K8s](/doc_images/k8s.png)

### Configure Node-Red flow in K8s container

16. Use http://159.xxx.xxx.xxx:30394 to access Node-Red flow editor.

17. Import `flows/k8s_simple_udp_listener_flow.json` (Import -> Clipboard and past the content of this `.json` file) and click on Deploy button. See that we use port number 1882 since this is the port which is mapped inside the container. 

    ![Simple UDP listener](/doc_images/k8s_simple_udp_listener_flow.png)


## [SerialDeviceApp](https://github.com/ptsk5/serial-device-app) configuration
1. Configure SerialDeviceApp examples (**app.js**, **app2.js** or **app3.js**) - line #13 and #14; and provide public IP address and UDP port as found above. 
    ```js
    #13| const _IP_ADDRESS = '159.xxx.xxx.xxx'; 
    #14| const _UDP_PORT   = '30662';
    ```
2. Run any of examples and verify that you can see incoming messages in the Node-Red debug console. 


## Internet of Things Platform Starter

 **Internet of Things Platform Starter** is a predefined set of Cloud Foundry modules (containers, services) which relates to the Internet of Things. This starter contains **Internet of Things Platform** service, **Cloudant** database, and **SDK for Node.js** container with Node-Red as a pre-installed application. 

 ![Internet of Things Platform Starter overview](/doc_images/cf.png)

1. Log in to IBM Cloud, find **Internet of Things Platform Starter** in the Catalog, and create an instance by providing at least the App name. 

2. Wait for the provisioning of all services to be finished. 

 ### Internet of Things Platform

3. Find the **Internet of Things Platform** you created in the previous steps as part of the **Internet of Things Platform Starter** package (in the Dashboard), and click on it to get to the _Manage_ detail of the service. 

4. Open the **Internet of Things Platform** management console by clicking on the **Launch** button.

5. Change to Devices -> Device Types and click on **+ Add Device Type** button. Create three device types by providing its name and type according to the following table:

    | Name              | Type    |
    |-------------------|---------|
    | GatewayDeviceType | Gateway |
    | ProxyDeviceType   | Device  |
    | NBIoTDeviceType   | Device  |

6. Change to Devices -> Browse and add click on **+ Add Device** button. Follow the wizard and create two devices according to the following table. Every time, you have to choose the correct Device Type and provide some Device ID. You do not need to modify any other optional parameters. In the Security step, choose own Authentication Token or let the system generate it for you. In any case, by clicking on the Done button at the end of the wizard, you can see the (generated) token, and you have to make a note of it. **Lost tokens cannot be recovered**.

    | Device ID         | Device Type    |
    |-------------------|---------|
    |  gatewayDevice    | GatewayDeviceType |
    |  proxyDevice      | ProxyDeviceType  |

7. We do not intentionally create any device ID of NBIoTDeviceType device type since the particular IDs will be generated automatically by the gatewayDevice from incoming messages. 

### Node-Red (as part of SDK for Node.js, not in K8s cluster)

We will create three main flows in Node-Red to realise: 
* **Gateway handler** - consumes data from proxyDevice, parses them, store them into Cloudant database, and sends them into the Internet of Things Platform on behalf of the particular devices. (_the flow of proxyDevice will be placed in the Node-Red deployed in K8s cluster and will be detailed at the end of this description_)
* **UI for device1** - example implementation of UI dashboard for device1.
* **Devices overview** - small UI overview which automatically detects and counts incoming messages and provides this information into a table. 

At the beginning, we need to initialise Node-Red and install required modules which we will use later.

8. Change back to the IBM Cloud console and find your application name among **Cloud Foundry Applications**.

9. Click on it to see the detail and follow the **Visit App URL** link on top of the page. 

10. Finish the initialisation wizard and click on **Go to your Node-Red flow editor** button.

11. Open Palette by choosing "Hamburger menu" in the right corner of the page and by clicking on **Manage palette**. In the opened window click on Install tab.

12. Find and install following modules:
    * **node-red-dashboard** - will be used for creating the dashboard,
    * **node-red-contrib-moment** - it can format time,
    * **node-red-contrib-ibm-watson-iot** - allows us to interact with the Internet of Thinkgs Platform as a gateway device.

#### Gateway handler

13. Import `flows/cf_gateway_handler_flow.json` (Import -> Clipboard and past the content of this `.json` file) and click on Deploy button.

    ![Gateway handler flow](/doc_images/cf_gateway_handler_flow.png)

14. You need to revisit the wiot node called `event` and click on a small pencil button next to Credentials to provide own information about Organization Id (update xxxxxx as marked in the picture below) and Auth Token for gatewayDevice. Update, Save and Deploy again.

    ![Watson IoT credentials](/doc_images/wiot_credentials.png)

15. At the same time, open `nbiot-data` Cloudant node and verify that the Service combo box lists the correct Cloudant database you have in your workspace (e.g. the one which was created as a part of **Internet of Things Platform Starter** package).


#### UI for device1

13. Import `flows/cf_ui_4_device1_flow.json` (Import -> Clipboard and past the content of this `.json` file) and click on Deploy button.

    ![UI for device1 flow](/doc_images/cf_ui_4_device1_flow.png)

#### Devices overview

13. Import `flows/cf_devices_overview_flow.json` (Import -> Clipboard and past the content of this `.json` file) and click on Deploy button.

    ![Devices overview flow](/doc_images/cf_devices_overview_flow.png)



## Revisit Node-Red flow in K8s container

To connect all the things together, we need to update the flow of Node-Red deployed in K8s container. This flow connects a node to the Internet of Things Platform which we did not have ready earlier. 

1. Use http://159.xxx.xxx.xxx:30394 to access Node-Red flow editor (see details about what is your correct URL above).

2. Remove the small flow we have already deployed.

3. Open Palette by choosing "Hamburger menu" in the right corner of the page and by clicking on **Manage palette**. In the opened window click on Install tab.

4. Find and install the following module:
    * **node-red-contrib-ibm-watson-iot** - allows us to send data into the Internet of Things Platform.

5. Import `flows/k8s_udp_to_wiotp_flow.json` (Import -> Clipboard and past the content of this `.json` file) and click on Deploy button. 

    ![Simple UDP listener](/doc_images/k8s_udp_to_wiotp_flow.png)

6. Similar to the previous case, double-click on the wiot node called `udpEvent` and continue by clicking on a small pencil button next to Credentials in order to provide own information about Organization Id (update xxxxxx) and Auth Token for proxyDevice. Update, Save and Deploy again.

## Final dashboard

Run the NB-IoT device to see the data in the cloud dashboard.

1. Connect Quectel GSM/NB-IoT EVB Kit to the computer / Raspberry Pi / ...

2. Run [SerialDeviceApp](https://github.com/ptsk5/serial-device-app) `app2.js` example (`node app2.js`) to see the dashboard for device1 filled with all possible information. `app.js` and `app3.js` examples send only example data sets (data1 and data2). They do not send information about signal quality and network statistics!

2. Find your URL of the Node-Red deployed as an application in the Cloud Foundry and replace `/red/` for `/ui/`, i.e. if your Node-Red is e.g. deployed on `https://<my app name>.eu-gb.mybluemix.net/red/`, open `https://<my app name>.eu-gb.mybluemix.net/ui/`.

    ![Dashboard - UI for device1](/doc_images/dashboard_ui_4_device1.png)

3. Open Devices overview page by choosing "Hamburger menu" in the left corner of the page and by clicking on Devices overview.

    ![Dashboard - Devices overview](/doc_images/dashboard_devices_overview.png)


## Next steps 

If you are aware of the IP address of the NB-IoT device (or at least of the IP address of the Telco provider's gateway which directs UDP datagrams to your application), you can create a flow which sends data back to your NB-IoT device by using udp output node.