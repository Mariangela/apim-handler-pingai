# Introduction

WSO2 API Manager is a full lifecycle API Management solution which has an API Gateway and a Microgateway. 

This explains how WSO2 API Manager plans to integrate with PingIntelligence and expose APIs protected with 
artificial intelligence (AI).

## WSO2 API Manager Extension with PingIntelligence

### What is PingIntelligence for APIs?
PingIntelligence for APIs uses artificial intelligence (AI) to expose active APIs, identify and automatically block cyber attacks on APIs and provide detailed reporting on all API activity. You can deploy the PingIntelligence solution on premises, in public clouds, or in hybrid clouds to monitor API traffic across the environment. PingIntelligence uses AI and machine learning models to detect anomalous API behavior, without relying on specifically defined policies or prior knowledge of attack patterns, in order to stop new and constantly changing attacks. In addition, PingIntelligence uses its continuous learning capabilities to become more accurate at identifying and blocking attacks over time. 

PingIntelligence for APIs can detect many types of cyberattacks, most of which are not visible to API teams today and can go undetected for very long times. 

[Read more about cyber attacks that can be detected by Ping Intelligence.](https://github.com/wso2-extensions/apim-handler-pingai/blob/master/DEVELOPER_GUIDE.md#types-of-attacks-pingintelligence-protects-against)

### How does integration happen?
The WSO2 API Manager extension for PingIntelligence uses a new custom handler (Ping AI Security Handler) when working with the WSO2 API Gateway data flow. After this handler receives a request from a client, a sideband call is sent to PingIdentity’s API Security Enforcer (ASE) with the client request metadata. The ASE responds after analyzing the metadata with an Artificial Intelligence Engine. 

If the response of ASE is 200 OK, the Ping AI Security Handler forwards the request and if the response is 403, it blocks the request.

![alt text](https://raw.githubusercontent.com/wso2-extensions/apim-handler-pingai/master/images/architecture.png)

## Quick Start Guide

### Prerequisites

- **Install Java 7 or 8.** 
(http://www.oracle.com/technetwork/java/javase/downloads/)
    
- **Install Apache Maven 3.x.x**
 (https://maven.apache.org/download.cgi#)

- **Install the latest WSO2 API Manager**
(https://wso2.com/api-management/)

    Installing WSO2 is very fast and easy. Before you begin, be sure you have met the installation prerequisites, 
    and then follow the [installation instructions for your platform](https://docs.wso2.com/display/AM260/Installing+the+Product).

- **Install the PingIntelligence software.**

    PingIntelligence 3.2.1 software is installed and configured. For installation of PingIntelligence software, 
    see the [manual or platform specific automated deployment guides](https://docs.pingidentity.com/bundle/PingIntelligence_For_APIs_Deployment_Guide_pingintel_32/page/pingintelligence_product_deployment.html).
- **Verify that ASE is in sideband mode.**
  
  Make sure that the ASE is in sideband mode by running the following command in the ASE command line:
    ```
   /opt/pingidentity/ase/bin/cli.sh status
   API Security Enforcer
   status                  : started
   mode                    : sideband
   http/ws                 : port 80
   https/wss               : port 443
   firewall                : enabled
   abs                     : enabled, ssl: enabled
   abs attack              : disabled
   audit                   : enabled
   sideband authentication : disabled
   ase detected attack     : disabled
   attack list memory      : configured 128.00 MB, used 25.60 MB, free 102.40 MB
    ```  
    
    If ASE is not in sideband mode, then stop the ASE and change the mode by editing the 
    */opt/pingidentity/ase/config/ase.conf* file. Set the mode as **sideband** and start the ASE.

- **Enable sideband authentication.**
  
  To ensure a secure communication between WSO2 Gateway and the ASE, enable sideband authentication by entering the following 
  command in the ASE command line:
   ```
    # ./bin/cli.sh -u admin –p admin enable_sideband_authentication 
   ```
   
- **Generate sideband authentication token.**

   A token is required for WSO2 Gateway to authenticate with ASE. To generate the token in the ASE, enter the following 
   command in the ASE command line:
   ```
   # ./bin/cli.sh -u admin -p admin create_sideband_token
   ```
   Save the generated authentication token for further use.
   
- **Add the certificate of the ASE to the WSO2 client keystore.**
   ```
    keytool -importcert -file <certificate_name>.cer -keystore <APIM_HOME>/repository/resources/security/client-truststore.jks -alias "Alias"
   ```
## Deploy the WSO2 extension with PingIntelligence

### For system admin

1. Download the extension and navigate to the **apim-handler-pingai** directory and run the following Maven command.
   ```
    mvn clean install
     ```
    org.wso2.carbon.apimgt.securityenforcer-\<version>.jar file can be found in **apim-handler-pingai/target** directory. 

2. Add the JAR file of the extension to the directory **<APIM_HOME>/repository/components/dropins**. 

3. Add the bare minimum configurations to the *api-manager.xml* within the tag \<APIManager>, which can be found in the
**<APIM_HOME>/repository/conf** directory.

    ```
    <PingAISecurityHandler>
        <OperationMode>async</OperationMode>
        <APISecurityEnforcer>
            <EndPoint>ASE_ENDPOINT</EndPoint>
            <ASEToken>SIDEBAND_AUTHENTICATION_TOKEN</ASEToken>
            <ModelCreationEndpoint>
                <EndPoint>ASE_REST_API_ENDPOINT</EndPoint>
                <AccessKey>ASE_REST_API_ACCESS_KEY</AccessKey>
                <SecretKey>ASE_REST_API_SECRET_KEY</SecretKey>
            </ModelCreationEndpoint>
       </APISecurityEnforcer>
    </PingAISecurityHandler>
   ```
    **Note:**
    - Select the Operation mode from **[sync](https://github.com/wso2-extensions/apim-handler-pingai/blob/master/DEVELOPER_GUIDE.md#sync-mode)**,
    **[async](https://github.com/wso2-extensions/apim-handler-pingai/blob/master/DEVELOPER_GUIDE.md#async-mode)**, and 
    **[hybrid](https://github.com/wso2-extensions/apim-handler-pingai/blob/master/DEVELOPER_GUIDE.md#hybrid-mode)**.
    If the mode is not set, the default mode is set as **async**. 
    - If you have not set the ModelCreationEndpoint configurations, you will need to manually create the ASE models.
    - Include the [sideband authentication token](https://github.com/wso2-extensions/apim-handler-pingai/blob/master/DEVELOPER_GUIDE.md#prerequisites)
     that you obtained from the ASE as the ASEToken.
     - For additional security you can [encrypt](https://github.com/wso2-extensions/apim-handler-pingai/blob/master/DEVELOPER_GUIDE.md#encrypting-passwords-with-cipher-tool) the SIDEBAND_AUTHENTICATION_TOKEN, ASE_REST_API_ACCESS_KEY, ASE_REST_API_SECRET_KEY.   

4. Update the **<APIM_HOME>/repository/resources/api_templates/velocity_template.xml** file in order to engage the handler to APIs. Add the handler class as follows inside the 
   *\<handlers xmlns="http://ws.apache.org/ns/synapse">* just after the foreach loop.
   ```
   <handler class="org.wso2.carbon.apimgt.securityenforcer.PingAISecurityHandler"/> 
   ```
  
5. Deploy WSO2 API Manager and access the management console: https://localhost:9443/carbon.
   
    Start the API Manager by going to <APIM_HOME>/bin using the command-line and executing wso2server.bat (for Windows) or wso2server.sh (for Linux.) 
    
6. Navigate to **Extensions** > **Configure** > **Lifecycles** and Click the **View/Edit** link corresponding to the 
*default API LifeCycle*.

7. Add a new execution for the **Publish** event under **CREATED** and **PROTOTYPED** states. 
Do not update the already existing execution for the publish event. Add a new execution.
    ```
    <execution forEvent="Publish" 
        class="org.wso2.carbon.apimgt.securityenforcer.executors.PingAIExecutor">
    </execution>
    ```
 
8. Add another execution for the **Retire** event under the **DEPRECATED** state.
   This deletes the model associated with the API in the ASE when the API is retired.
    ```
    <execution forEvent="Retire" 
        class="org.wso2.carbon.apimgt.securityenforcer.executors.PingAIExecutor">
    </execution>
    ```
     
### For the API Publisher

**For new APIs**

- After the API is successfully created and the life cycle state changes to **PUBLISHED**,
 a new model is created in the ASE for the API and the handler is added to the data flow. 
 When the API state changes to **RETIRED**, the model will be deleted.

**For existing APIs**

- The recommended method is to create a [new version](https://docs.wso2.com/display/AM260/Quick+Start+Guide#QuickStartGuide-VersioningtheAPI) 
for the API with PingIntelligence enabled.

    *Although changing the status of a live API is not recommended, when an API is republished, it updates the Synapse config with the handler and 
    by demoting to CREATED or PROTOTYPED state and changing the life cycle back to PUBLISHED state 
   will create a new model for API in the ASE.*


**Note:**
By default, PingIntelligence is enabled in all APIs that are published with an individual AI model. 
However, if needed you can configure PingIntelligence to be [applied only for selected APIs.](https://github.com/wso2-extensions/apim-handler-pingai/blob/master/DEVELOPER_GUIDE.md#add-the-policy-only-for-selected-apis)


### Developer Guide

For more information, see the [Developer Guide](https://github.com/wso2-extensions/apim-handler-pingai/blob/master/DEVELOPER_GUIDE.md).
