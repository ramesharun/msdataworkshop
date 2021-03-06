"Intelligent Event-driven Stateful Microservices with Helidon and Autonomous Database on OCI" 

Please send any questions or comments to Paul Parkinson and/or Helidon Slack at https://helidon.slack.com/archives/CCS216A5A

The following architecture can all be created entirely free using OCI, ATP, and Helidon. 

The workshop is designed to be modular and dynamic such that it is possible to deploy only the aspects (ie infrastructure and services) of interest.

![demo architecture](images/demo-arch.png) 
![helidon](images/helidon.png) 


## Task 1 (Create OCI account, OKE cluster, ATP databases) ##
![OCI-AOK-ATP-setup](images/OCI-OKE-ATP-setup.png) 
   - Estimated task time 20 minutes
   - Video walk through: https://www.youtube.com/watch?v=0LeyGPw2vAA
   - Get (free) OCI account and tenancy 
        - https://myservices.us.oraclecloud.com/mycloud/signup
        - note tenancy ocid, region name, user ocid
   - Create user api key and note the private key/pem location, fingerprint, and passphrase (be sure to use a passphrase)
        - https://docs.cloud.oracle.com/en-us/iaas/Content/Functions/Tasks/functionssetupapikey.htm
   - Create compartment
        - https://docs.cloud.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcompartments.htm?Highlight=creating%20a%20comparment
        - https://oracle-base.com/articles/vm/oracle-cloud-infrastructure-oci-create-a-compartment#create-compartment
   - Create OCIR repos and auth key
        - Create a meaningful repos name such as `paul.parkinson/msdataworkshop` and note the repo-name you've created as it will be used in Task 4
        - https://docs.cloud.oracle.com/en-us/iaas/Content/Registry/Tasks/registrycreatingarepository.htm
   - Create OKE cluster
        - https://docs.cloud.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke.htm
        - https://docs.cloud.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengaccessingclusterkubectl.htm
        - https://docs.cloud.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengdownloadkubeconfigfile.htm
   - Create 2 ATP-S pdbs named `orderdb` and `inventorydb` (for order and all other services)
        - If the pdbs are not named `orderdb` and `inventorydb`, insure the default exported values for ORDER_PDB_NAME and ORDER_PDB_NAME are changed (this is covered in later step)
        - Select the license included option for license.
        - https://docs.oracle.com/en/cloud/paas/autonomous-data-warehouse-cloud/tutorial-getting-started-autonomous-db/index.html 
        - Note the ocid, compartmentId, name, and admin pw of the databases
        - Download the regional wallet (connection info) and note the wallet password (this is optional depending on setup - todo elaborate)

## Task 2 (Use Cloud Shell to access OKE cluster and create `msdataworkshop` namespace) ##
![OCI-cloudshell-ATP-setup](images/cloudshell.png) 
   - Estimated task time 2 minutes
   - Video walk through: https://www.youtube.com/watch?v=vYD1s6c3a8w
   - Enter Cloud Shell and issue command to export kubeconfig for the OKE cluster created
   - Related blog with quick instructions here: https://blogs.oracle.com/cloud-infrastructure/announcing-oracle-cloud-shell
   - Verify OKE access using command such as `kubectl get pods --all-namespaces`
   - Create `msdataworkshop` namespace using command `kubectl create ns msdataworkshop`
    
## Task 3 (Create github account and build microservice image) ##
   - Estimated task time 2 minutes
   - Video walk through: https://www.youtube.com/watch?v=6g4c2yjbTPg
   - Optionally (if planning to make modifications, for example) 
        - Create github account if needed
        - Fork `https://github.com/paulparkinson/msdataworkshop.git` 
   - From Cloud Shell...
   - Run `git clone https://github.com/paulparkinson/msdataworkshop.git`
        - or if using a fork then `git clone` that fork instead
   - `cd msdataworkshop/frontend-helidon`
   - Run `mvn clean install`

## Task 4 (Push image, deploy, and access microservice) ##
![first-microservice-deployed-static](images/first-microservice-deployed-static.png) 
   - Estimated task time 8 minutes
   - Video walk through: https://www.youtube.com/watch?v=I10NLrAYjIM
   - From Cloud Shell...
   - Login to OCIR and verify docker
        - https://docs.cloud.oracle.com/en-us/iaas/Content/Registry/Tasks/registrypushingimagesusingthedockercli.htm
        - Run `docker login <region-key>.ocir.io -u <tenancy-namespace>/<username> -p <authtoken>` 
            - `<tenancy-namespace>` is the Object Storage Namespace found under tenancy information
            - example `docker login us-phoenix-1.ocir.io -u ax2mkasdfkkx/msdataworkshopuser -p Q:4qXo:7ADFaf9KZddZQ`
        - Verify with `docker image` command
   - For convenience, vi ~/.bashrc, append the following lines 
        (substituting MSDATAWORKSHOP_NAMESPACE, MSDATAWORKSHOP_LOCATION, DOCKER_REGISTRY, and *PDB_NAME values as appropriate) 

         export MSDATAWORKSHOP_NAMESPACE=msdataworkshop
         export MSDATAWORKSHOP_LOCATION=~/msdataworkshop
         source $MSDATAWORKSHOP_LOCATION/shortcutaliases
         export PATH=$PATH:$MSDATAWORKSHOP_LOCATION/utils/
         export DOCKER_REGISTRY="<region-key>.ocir.io/<tenancy-namespace>/<repo-name>"
         export ORDER_PDB_NAME=orderdb
         export INVENTORY_PDB_NAME=inventorydb
   - `source ~/.bashrc`
   - `cd $MSDATAWORKSHOP_LOCATION/frontend-helidon`
   - Run `./build.sh` to build the frontend-helidon image and push it to OCIR
   - Mark the image as public in OCIR via Cloud Shell (this avoids the need to do `docker login` in the deployment yaml or git CI/CD)
   - Run `./deploy.sh` to create deployment and service in the msdataworkshop namespace 
   - Check frontend pod is Running by using `kubectl get pods --all-namespaces` or the `pods` shortcut command
   - Check frontend loadbalancer address using `kubectl get services --all-namespaces`  or use `services` shortcut command
   - Access frontend page 
        - via frontend LoadBalancer service, eg http://129.146.99.99:8080
        - or, if the service has been modified to use NodePort instead of LoadBalancer...
            - `kubectl port-forward [frontend pod] -n msdataworkshop 8080:8080`
            - and access http://localhost:8080
![appfrontend](images/appfrontend.png)             
   - Run `cd $MSDATAWORKSHOP_LOCATION ; ./build.sh ` to build and push all of the rest of the microservices
   - Mark all of the images as public in OCIR via Cloud Shell as done with the frontend image

## Task 5 (Setup OCI Open Service Broker) ##
   - Estimated task time 5 minutes
   - Video walk through: https://www.youtube.com/watch?v=cb8N1TzNgsk
   - `cd $MSDATAWORKSHOP_LOCATION/osb-atp-and-oss`
   - Set ocid, etc. values in `setupOSB.sh` and run `./setupOSB.sh`
   - Refererences... 
        - https://github.com/oracle/oci-service-broker/blob/master/charts/oci-service-broker/docs/installation.md
        - https://www.youtube.com/watch?v=qW_pw6Nd5hM
   - Troubleshooting... 
        - If oci-service-broker status continues to be "ErrorFetchingCatalog”, see OSB Troubleshooting document for details on debugging common and known issues: https://github.com/oracle/oci-service-broker/blob/master/README.md
   
## Task 6 (Using OCI service broker, create binding to 2 existing ATP instances) ##
   - Estimated task time 5 minutes
   - Video walk through: https://www.youtube.com/watch?v=EGO-bLHrhv0
   - `cd $MSDATAWORKSHOP_LOCATION/osb-atp-and-oss`
   - Set ocid and password values in `setupATP.sh` and run `./setupATP.sh`
   - References...
        - https://github.com/oracle/oci-service-broker/blob/master/charts/oci-service-broker/docs/atp.md
        - https://www.youtube.com/watch?v=qW_pw6Nd5hM
  
## Task 7 (Verify and understand ATP connectivity via Helidon microservice deployment in OKE) ##
![helidon](images/atpfromhelidonsteps.png) 
   - Estimated task time 5 minutes
   - Video walk through: https://www.youtube.com/watch?v=k44qrpdohTM
   - Notice [atpaqadmin-deployment.yaml](atpaqadmin/atpaqadmin-deployment.yaml) wallet, secret, decode initcontainer, etc. 
   - Notice `atp*` references in [microprofile-config.properties](atpaqadmin/src/main/resources/META-INF/microprofile-config.properties) 
   - Notice @Inject dataSources in [ATPAQAdminResource.java](atpaqadmin/src/main/java/oracle/db/microservices/ATPAQAdminResource.java)
   - `cd $MSDATAWORKSHOP_LOCATION/atpaqadmin`
   - Run `./deploy.sh` to create deployment and service
   - Run `msdataworkshop` command to verify existence of deployment and service and verify pod is in Running state
   - Open the frontend microservice home page and hit the `testdatasources` button 
   - Troubleshooting... 
        - Look at logs... `kubectl logs [podname] -n msdataworkshop`
        - If no request is shown in logs, try accessing the pod directly using port-forward
            - `kubectl port-forward [atpadmin pod] -n msdataworkshop 8080:8080`
            - http://localhost:8080/test
   - Troubleshooting... 
        - If any of the pods' status is not Running...
            - check the pod's logs using `logpod admin` (or `order`, `inventory`, `supplier`, ...) and 
            - use `describepod admin` (or `order`, `inventory`, `supplier`, ...) utility to get more details.
            - If you see the errors similar to the ones below, re-run `osb-atp-and-oss/setupATP.sh` and run `deletepod` for each deployment (eg `deletepod admin ; deletepod order ; deletepod inventory; deletepod supplier`) in order to restart the pods and pick up the secrets/bindings:
              •	Warning  FailedMount  8s (x8 over 72s)  kubelet, 10.0.10.3  MountVolume.SetUp failed for volume "creds-raw-inventory" : secret "atp-demo-binding-inventory" not found
              •	Warning  FailedMount  8s (x8 over 72s)  kubelet, 10.0.10.3  MountVolume.SetUp failed for volume "creds-raw-order" : secret "atp-demo-binding-order" not found

              
## Task 8 (Setup DB links between ATP PDBs, AQ, and Queue propagation) ##
![dbadminservice](images/dbadminservice.png)           
   - Estimated task time 8 minutes
   - Video walk through: https://www.youtube.com/watch?v=Nb6i5XQgdqA
   - Download connection information zip for ATP instances from console.
   - Upload cwallet.sso to objectstore, obtain and note pre-authorized URL for cwallet.sso
        - (this is for convenience, alternatively a DBMS_CLOUD.CREATE_CREDENTIAL can be used to create a credential that is then used to execute GET_OBJECT in PropagationSetup.java)
   - `cd $MSDATAWORKSHOP_LOCATION/atpaqadmin`
   - Edit `atpaqadmin-deployment.yaml` and provide values in the section marked with `PROVIDE VALUES FOR THE FOLLOWING...`
   - Run `./redeploy.sh` 
   - Run `pods` and verify the atpaqadmin pod is in running state
   - Open the frontend microservice home page and hit the following buttons in order  
        - `createUsers`, `createInventoryTable`, `createDBLinks`, `setupTablesQueuesAndPropagation`
   - If it is necessary to restart or rerun the process or to clean the database...
        - hit the `unschedulePropagation` button only if `setupTablesQueuesAndPropagation` was run and then hti the `deleteUsers` button
        - it may be necessary to run `deletepod admin` first as there may be open connections that need to be be released.
   - If any changes are made to src code or deployment, simply run `./build.sh ; ./redeploy.sh` to rebuild and redeploy
   - Troubleshooting... 
        - Look at logs... `kubectl logs [podname] -n msdataworkshop`
        - If `ORA-12529: TNS:connect request rejected based on current filtering rules` is shown in logs, re-create the pdb wallets. 
            - Insure they are regional such that tnsnames.ora contains entries for both pdbs and the cwallet.sso applies to both pdbs and not just one.
        - If messages sent by order service are not received by inventory service (or vice-versa) hit the `enablePropagation` button on the frontend. This may occur if a pdb has had maintenace performed on it and/or moved.
                                    
## Task 9 (Demonstrate Converged database (relational, JSON, spatial, etc.), Event-driven Order/Inventory Saga, Event Sourcing, CQRS, etc. via Order/Inventory store application) ##
![orderinventoryapp-microservices](images/orderinventoryapp-microservices.png)  
![demo ERD](images/demo-erd.png) 
   - Estimated task time 12 minutes
   - Video walk through: https://www.youtube.com/watch?v=B-lof08Ip38
   - `cd $MSDATAWORKSHOP_LOCATION/order-helidon ; ./build.sh ; ./deploy.sh`
   - `cd $MSDATAWORKSHOP_LOCATION/inventory-helidon ; ./build.sh ; ./deploy.sh`
   - `cd $MSDATAWORKSHOP_LOCATION/supplier-helidon-se ; ./build.sh ; ./deploy.sh`
   - Open the frontend microservice home page
   - Hit the `listenForMessages` button to allow inventory service to listen for events (fyi this will not be removed and not necessary in future version of the app)
   - Check inventory for a given item such as a cucumbers by hitting the `getInventory` button and insure (`removeInventory` if necessary) no inventory exists
   - Hit the `placeorder` button for that item (eg `veggie: cucumbers` `orderid: 66`) and notice the initial/pending state of the order
   - Hit the `showorder` button and notice failed order due to choreography saga compensation
   - Hit the `addInvetory` button for the item 
   - Hit the `placeorder` button with a different orderid (or `deleteorder` the previous/failed orderid)
   - Hit the `showorder` button and notice successful order due to choreography saga compensation. Also notice CQRS
   - This demonstrates atomic exactly once event-driven communication over AQ, 
   - This also demonstrates CQRS as the command is executed on the database while the query is derived from events received.
   - If any changes are made to src code or deployment, simply run `./build.sh ; ./redeploy.sh` to rebuild and redeploy
   - Notice Order and Inventory service src demonstrating transactional AQ
   - Docs/references...
        - JSON-P (Processing) https://javaee.github.io/jsonp/
        - JSON-B (Binding) https://javaee.github.io/jsonb-spec/
        - Helidon (MP/MicroProfile standard and SE) http://helidon.io
        - Microservices on Helidon (blog including JSON examples) https://www.baeldung.com/microservices-oracle-helidon
        - Oracle SODA (Simple Oracle Document Access) https://docs.oracle.com/en/database/oracle/simple-oracle-document-access/
   
## Task 9.5 Demonstrate spatial service running on Weblogic (operator) via order delivery routing service) ##
   - https://github.com/nagypeter/weblogic-operator-tutorial/blob/master/tutorials/domain.home.in.image_short.md
   - https://blogs.oracle.com/weblogicserver/easily-create-an-oci-container-engine-for-kubernetes-cluster-with-terraform-installer-to-run-weblogic-server
   - https://www.oracle.com/middleware/technologies/weblogic.html
   
## Task 10 (Metrics) ##
   - Notice io.helidon.metrics dependency in pom.xml and @Counted in [OrderResource.java](order-helidon/src/main/java/io/helidon/data/examples/OrderResource.java)
   - Hit the `metrics` button and notice the various metrics (in prometheus format)
   - These metrics can then be viewed in Grafana

## Task 11 (Helidon/OKE health liveness/readiness)  ##
   - Estimated task time 7 minutes
   - Video walk through: https://www.youtube.com/watch?v=56Xk65qgP3U
   - eg order service is not ready until some data load (from view or eventsourcing or lazily) is done
   - Notice io.helidon.health dependency in pom.xml and [OrderLivenessHealthCheck.java](order-helidon/src/main/java/io/helidon/data/examples/OrderLivenessHealthCheck.java)
   - Notice livenessProbe in helidon-order-deployment.yaml
   - documentaion references...
       - https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
       - https://github.com/oracle/helidon/blob/master/docs/src/main/docs/health/02_health_in_k8s.adoc
       - https://github.com/oracle/helidon/blob/master/docs/src/main/docs/guides/07_health_se_guide.adoc
       - https://dmitrykornilov.net/2019/08/08/helidon-brings-microprofile-2-2-support/
   - If any changes are made to src code or deployment, simply run `./build.sh ; ./redeploy.sh` to rebuild and redeploy

## Task 12 (Tracing) ##
   - Install Jaeger in OKE using `kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-kubernetes/master/all-in-one/jaeger-all-in-one-template.yml -n msdataworkshop`
   - Notice tracing properties such as `tracing.host=jaeger-collector.msdataworkshop` (and modify them if defaults were not used during Jaeger install) in [microprofile-config.properties](order-helidon/src/main/resources/META-INF/microprofile-config.properties)
   - Notice io.helidon.tracing dependency in pom.xml and @Traced in [OrderResource.java](order-helidon/src/main/java/io/helidon/data/examples/OrderResource.java)
   - Place an order using the application if not already done.
   - By default the Jaeger install will create a load balancer and the address can be found by issuing `kubectl get services -n msdataworkshop | grep jaeger-query`
   - Open Jaeger console (eg http://[loadbalancer-address]:80 ) and observe traces. Select the Helidon MP order service under the `services` dropdown

## Task 13 (Demonstrate OKE horizontal pod scaling) ##
   - Estimated task time 5 minutes
   - Video walk through: https://www.youtube.com/watch?v=pII2yS7C0r8
   - `cd $MSDATAWORKSHOP_LOCATION`
   - Run `./installMetricsServer.sh` 
   - `cd $MSDATAWORKSHOP_LOCATION/order-helidon`
   - uncomment `resources` section `Task 12` in `order-helidon-deployment.yaml`
   - Run `./redeploy.sh`
   - Run `toppod order` and notice CPU and memory usage
   - Run `kubectl autoscale deployment order-helidon --cpu-percent=50 --min=1 --max=2 -n msdataworkshop`
   - Run `hpa` and notice output
   - Open the frontend microservice home page and hit the `startCPUStress` button
   - increase cpu, notice cpu increase and scale to 2 pods
   - `kubectl delete hpa order-helidon -n msdataworkshop` to clean up
   - alternate demo...
       - `kubectl run oraclelinux77-hpa-demo --image=phx.ocir.io/oraclegilsonmel/oraclelinux77-demo:latest --requests=cpu=200m --limits=cpu=500m tailf /dev/null -n msdataworkshop`
       - `pods | grep linux`
       - `kubectl autoscale deployment oraclelinux77-hpa-demo --cpu-percent=50 --min=1 --max=10 -n msdataworkshop`
       - `hpa ; toppod linux ; k get deployment oraclelinux77-hpa-demo -n msdataworkshop  ; pods |grep linux ;echo ----------------`
       - `podshell linux`
       - `stress --cpu 4 &` and exit
       - `hpa ; toppod linux ; k get deployment oraclelinux77-hpa-demo -n msdataworkshop  ; pods |grep linux ;echo ----------------` 
       - repeated above command as necessary noting changes and additional pods
       - cleanup...
       - `k delete deployment oraclelinux77-hpa-demo -n msdataworkshop`
       - `k delete hpa oraclelinux77-hpa-demo -n msdataworkshop`
   - If any changes are made to src code or deployment, simply run `./build.sh ; ./redeploy.sh` to rebuild and redeploy
  
## Task 14 Polyglot

    # Review Polyglot Code
    ## Introduction

    This lab will show you how to switch the Inventory microservice to a Python, Node.js or Java Helidon SE implementation while retaining the same application functionality.

    ### Objectives
    -   Undeploy the existing Java Helidon MP Inventory microservice
    -   Deploy an alternate implementation of the Inventory mmicroservice and test the application functionality

    ### What Do You Need?

    This lab assumes you have already completed the following labs:
    - Sign Up for a Free Tier
    - Setup OCI, OKE, ATP and Cloud shell
    - Build the GitHub code and deploy the microservices
    - Setup Service broker and Messaging

    ## **STEP 1**: Undeploy the Java Helidon MP Inventory Microservice

    1. To undeploy the Inventory Helidon service, open the Cloud Shell and go to the
    inventory-helidon folder, using the following command.

    ```
    <copy>cd $MSDATAWORKSHOP_LOCATION/inventory-helidon ; ./undeploy.sh</copy>
    ```

    ## **STEP 2**: Deploy and alternate implementation of the Inventory Microservice

    In this step you can choose between three different implementations of the Inventory Microservice - 1. Java Helidon MP, 2. Python, or 3. Node.js

    1. To build and deploy the Java Helidon SE implementation of the Inventory Microservice, open the Cloud Shell and execute the following command the following command.
    ```
    <copy>cd $MSDATAWORKSHOP_LOCATION/inventory-helidon-se ; ./build.sh ; ./deploy.sh</copy>
    ```

    TODO

    Once the image has been deployed in a pod, you should see the following message.

    TODO

    2. To build and deploy the Python implementation of the Inventory Microservice, open the Cloud Shell and execute the following command the following command.
    ```
    <copy>cd $MSDATAWORKSHOP_LOCATION/inventory-python ; ./build.sh ; ./deploy.sh</copy>
    ```

    TODO

    Once the image has been deployed in a pod, you should see the following message.

    TODO

    3. To build and deploy the Node.js implementation of the Inventory Microservice, open the Cloud Shell and execute the following command the following command.
    ```
    <copy>cd $MSDATAWORKSHOP_LOCATION/inventory-nodejs ; ./build.sh ; ./deploy.sh</copy>
    ```

    TODO

    Once the image has been deployed in a pod, you should see the following message.

    TODO

    ## **STEP 3**: Verify Application Functionality

    1. Check the inventory of a given item such as cucumbers, by typing `cucumbers`
    in the veggie field and clicking **getInventory**. You should see the inventory
    count result 0.

   ![](images/ea46ee63349f987bd43f772ed6562a87.png " ")

    2. (Optional) If for any reason you see a different count, click **removeInventory** to bring back the count to 0.

    3. Let’s try to place an order for cucumbers by clicking **place order**.

   ![](images/3ed8a96fec2a7ed044dda26b67865df2.png " ")

    4. To check the status of the order, click **showorder**. You should see a failed
    order status.

   ![](images/657d263f888691e7f1070d92201757b7.png " ")

    This is expected, because the inventory count for cucumbers was 0.

    5. Click **addInventory** to add the cucumbers in the inventory. You
    should see the outcome being an incremental increase by 1.

   ![](images/2acf1d8f9634c598b44b5dd0f3815457.png " ")

    6. Go ahead and place another order by clicking **place order**, and then click
    **showorder** to check the order status.

   ![](images/173839f1dd7c467a9706e551433af67b.png " ")

   ![](images/4916798cb22e9cd8a7dfa4d8dc01c5b9.png " ")

    The order should have been successfully placed, which is demonstrated with the order status showing success.

## Task 15 (LRA) ##
   - https://medium.com/oracledevs/long-running-actions-for-microprofile-on-helidon-data-integrity-for-microservices-2bd4d14fe955
   - LRA + Camunda 
   - FA
   
## Task 16 (JPA) ##
   - JPA/JTA, though not directly related potentially OpenAPI/swagger, etc. as well
   - (security, vault, etc - story around relation of vault and OSB being discussed now)

## Task 17 (OSS streaming service) ##
   - Insure Task 4 is complete and refer to https://github.com/oracle/oci-service-broker and specifically...
        - https://github.com/oracle/oci-service-broker/blob/master/charts/oci-service-broker/docs/oss.md
   - In Cloud Shell and streaming policy
        - add a group for user if one does not exist
        - add policy for that group to allow streaming (eg name `StreamingPolicy`, description `Allow to manage streams`)
            - Policy statement `Allow group <SERVICE_BROKER_GROUP> to manage streams in compartment <COMPARTMENT_NAME>`
            - eg `Allow group msdataworkshop-admins to manage streams in compartment msdataworkshop-sandbox)`' 
   - todo... Currently hitting some issues with the following and resorting to manually setting 
   - cd to `oci-service-broker` directory such as oci-service-broker-1.3.3
   - `cp samples/oss/create-oss-instance.yaml create-oss-instance-order.yaml`
   - Modify `create-oss-instance-order.yaml` 
        - change name to `teststreamorder` provide compartmentID and specify `1` partition
   - Run `kubectl create -f create-oss-instance-order.yaml -n msdataworkshop`
   - `cp samples/oss/create-oss-binding.yaml create-oss-binding-order.yaml`
   - Modify `create-oss-binding-order.yaml` 
        - change name to `test-stream-binding-order` and change instanceRef name to `teststream-order'
   - Run `kubectl create -f create-oss-binding-order.yaml -n msdataworkshop`
   - Run `kubectl get secrets test-stream-binding-order -o yaml -n msdataworkshop`
   - Demonstrate streaming orders in frontend app by hitting `producerstream` button
   - If any changes are made to src code or deployment, simply run `./build.sh ; ./redeploy.sh` to rebuild and redeploy
    
## Task 18 (HA) ##
    - Outage 
    - chaos test
   
## Task 19 (graph) ##
    - social

## Task 20 (sharded queues) ##
    - 

## Task 21 (migration from SOA) ##
    -
    - OWSM policy to service mesh side-car pattern, istio/linkerd

## Task 22 (apex)  ##
    - data-driven, no code, analytics of orders etc. or where order is eg

## Task 23 (data flow)  ##
    - Spark : IoT of drones

## Task 24 (data science)  ##
    - enables machine learning models on Oracle Cloud: predictive analytics of orders to inventory/delivery locations

## Task 25 (data catalog)  ##
    - enables data consumers to easily find data assets across the enterprise using an inventory of data assets: analytics report of order info from streaming + atp