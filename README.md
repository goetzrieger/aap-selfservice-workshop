# Ansible Automation Platform (AAP) - Self Service Portal

This workshop will cover setting up AAP and the new AAP Self-Service Portal on Openshift. A reasonably well-equipped Single Node OpenShift Instance (e.g. 32 cores/128 GB RAM) should suffice.

NOTE:
====
For Red Hat associates: This lab guide is based on the RHDP item **OpenShift GitOps Blank Environment**
====

## AAP Installation on OpenShift

To get going we need an Ansible Automation Platform 2.6 instance, so why not quickly deploy one on OpenShift. This is an easy task since it's done via an Operator:

* Log in to Openshift as cluster-admin 
* Install the Ansible Automation Platform Operator, leave all settings on default
* Wait until the Operator installation has finished
* Go to **Installed Operators** and click the **Ansible Automation Platform** Operator

You are now ready to install an instance of AAP. As you can see the Operator provides a good number of APIs you can use to manage a lot of the aspects of AAP.
The easiest way is to create an instance of the tile **Ansible Automation Platform**, this deploys the complete platform (Automation Controller, Automation-Hub and EDA).

* On the tile, click **Create instance**
* To get full control of the process, switch from **Form view** to **YAML view**
* In this YAML snippet replace `<read-write-many-storage-class>` with the RWX storage class `ocs-external-storagecluster-cephfs`, then replace the **spec** section with this content:

```
spec:
  database:
    resource_requirements:
      requests:
        cpu: 200m
        memory: 512Mi
    storage_requirements:
      requests:
        storage: 100Gi

  controller:
    disabled: false

  eda:
    disabled: false

  hub:
    disabled: false
    storage_type: file
    file_storage_storage_class: <read-write-many-storage-class>
    file_storage_size: 10Gi
```
Now wait until the AAP instance has been deployed by the Operator. After all ressources have been deployed, look up the Automation Controller URL and access it.

Get the admin password from the OpenShift secret:
* In the OpenShift UI, make sure you are in **Project** `AAP`, then go to **Workloads->Secrets** and copy the password from the secret `<instance name>-admin-password`.
* Login to the AAP UI as user `admin` and the password you extracted from the secret.

You should now see the dialog in AAP asking for a subscription, you can supply it either by a Subscription Manifest provided by your facilitator or by logging into your Red Hat account that must contain an AAP subscription. 
After providing an AAP Subscription you should be greeted with a shiny new AAP 2.6 UI!

## Installation of AAP Self-Service Automation Portal

### Preparation: OAuth Application and Tokens

The Self-Service Portal is a customized version of Red Hat Developer Hub, which is based on the Backstage project. It includes a number of custom plugins to provide the features for automation self-service.
Currently it's only available on OpenShift, but a RHEL version is on the roadmap.

There are a good number of steps required to install it so let's go ahead. The full installation documentation can be found [here](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_self-service_automation_portal).

The Self-Service Portal is installed via a Helm Chart. The first step is to create an OAuth application in AAP for authentication:

* You should be logged i nto the AAP UI as user admin
* Navigate to **Access Management -> OAuth Applications**
* Click **Create OAuth Applications**
* Complete the fields in the form:
  * **Name:** selfservice
  * **Organization:** default
  * **Authorization grant type:** Authorization code
  * **Client type:** Confidential
  * **Redirect URIs:** https//www.example.com, this will be replaced later

Now a window `OAuth Application Secrets` pops up, it's important to store the `Client ID` and `Client Secret` tokens!

The next step is to configure AAPs OAuth settings to enable OAuth token creation for external users:

* In AAP, select **Settings -> Platform gateway**
* Click **Edit platform gateway settings**
* Find the **Allow external users to create OAuth2 tokens** setting and set it to `Enabled`
* Click **Save Platform Gateway Settings**

In addition you must create a token in Ansible Automation Platform. The token is used in an OpenShift secret for Ansible Automation Platform authentication, as admin user do:

* Navigate to **Access Management -> Users**
* Select the `admin` user
* Select the **API Tokens** tab
* Click **Create API Token**
* Select your OAuth application `selfservice`. In the Scope menu, select `Read`.
* Click **Create Token** to generate the token.
* From the **Token information** window save the new token

The token is used in an OpenShift secret that is fetched by the Helm chart.

### Create OpenShift Project, the Plug-In Registry and Secrets

First create an OpenShift project where the Self-Service Portal will be installed to. Feel free to use the `oc` tool or in the OpenShift UI do:

* **Home -> Projects**
* Click **Create Project**
* **Name:** selfservice
* Click **Create**

The Self-Service Portal is based on Red Hat Developer Hub (RHDH), which is plug-in based and highly customizable. We have to integrate the custom plug-ins which provide the self-service functionality, as a plug-in registry. At the moment the plug-ins are delivered as a tar-file.

The first step is to download the tar-file from the Red Hat Customer Portal. In a browser, go to `access.redhat.com`, log in and navigate to **Downloads -> Red Hat Ansible Automation Platform**. On the `Product Software` click **Download now** next to `Ansible self-service automation portal Setup Bundle`.

[NOTE]
====
The next steps include running the `oc` command on a Linux system. You can either use your own laptop or SSH into the bastion host when doing this lab on the Red Hat Demo Platform. WSL on Windows might work, too... 
====

After you downloaded the tar-file with the plugins, create a directory to store the files. Then set an environment variable pointing to the location, e.g.:

```
$ mkdir /path/to/<automation-portal-plugins-local-dir-changeme>
$ export DYNAMIC_PLUGIN_ROOT_DIR=/path/to/<automation-portal-plugins-local-dir-changeme>
```

Extract the self-service-automation-portal-plugins-<version-number>.tar.gz contents to $DYNAMIC_PLUGIN_ROOT_DIR:

```
$ tar --exclude='*code*' -xzf self-service-automation-portal-plugins-x.y.z.tar.gz -C $DYNAMIC_PLUGIN_ROOT_DIR
```

With these files you will now setup the plug-in registry on OpenShift. First log in with `oc` to your OpenShift instance, the easiest way is to copy the login command from the OpenShift UI: Drop-down **Admin -> Copy login command**. Now create the registry:

* Open your OpenShift project: `$ oc project selfservice`

Run the following commands to create a plugin registry build in in your OpenShift project.

```
oc new-build httpd --name=plugin-registry --binary
oc start-build plugin-registry --from-dir=$DYNAMIC_PLUGIN_ROOT_DIR --wait
oc new-app --image-stream=plugin-registry
```

You can verify that the plugin-registry deployed correctly in the OpenShift Container Platform web console, or you can use a CLI command. Run the following command from a terminal to verify that the plugin-registry deployed correctly:

```
$ oc exec $(oc get pods -l deployment=plugin-registry -o jsonpath='{.items[0].metadata.name}') -- ls -l /opt/app-root/src
```

Confirm that the following required TAR files are in the plugin registry:

```
ansible-plugin-scaffolder-backend-module-backstage-rhaap-dynamic-x.y.z.tgz
ansible-backstage-plugin-auth-backend-module-rhaap-provider-dynamic-x.y.z.tgz
ansible-backstage-plugin-catalog-backend-module-rhaap-dynamic-x.y.z.tgz
ansible-plugin-backstage-self-service-dynamic-x.y.z.tgz
```

### Creating Ansible Automation Platform authentication secrets 

Before you can install the Self-Service Portal the OAuth client id and secret and the AAP token must be created as an OpenShift secret to make them available.

* In the OpenShift UI switch to the `selfservice` Project
* In the side menu, go to **Workloads -> Secrets**
* Click **Create -> Key/Value secret**
* Create a secret named `secrets-rhaap-portal`
* Now add four key/value pairs to the Secret:
  * Key: aap-host-url
    Value: <Ansible Automation Platform instance URL with leading https://>
  * Key: oauth-client-id
    Value: Ansible Automation Platform OAuth client ID you saved earlier
  * Key: oauth-client-secret
    Value: Ansible Automation Platform OAuth client secret value you saved earlier
  * Key: aap-token
    Value: Token for Ansible Automation Platform user authentication (must have write access) you saved earlier

### Installing the Self-Service Portal

Now that you have:

* created a project for self-service automation portal
* created a plugin registry in your project
* set up secrets for Ansible Automation Platform authentication

You can finally go ahead and install the software using a Helm Chart:

* In the OpenShift UI side menu go to **Helm -> Releases** 
* Click the `Browse the catalog to discover available Helm Charts` link
* Search for `auto` and click the `AAP self-service automation portal` tile
* Click **Create** 
* In the `Create Helm Release` page go to the YAML view and locate the `clusterRouterBase` key
* Replace the placeholder value with the base URL of your OpenShift instance.

NOTE:
====
The base URL is the URL portion of your OpenShift URL that follows https://console-openshift-console, for example apps.cluster-wxyz.dynamic.redhatworkshops.io
====

* Click **Create**

This will take you to the Topology view of the Self-Service Portal application. Note that the application consists of a PostGres DB, the plug-in registry you created earlier and the Portal itself. It will take some minutes for all components to start, when all components have switched color from light blue to dark blue you are good to go!

### Access the Self-Service Automation Portal

There is just one last step before you can start to use the portal... remember when you created the OAuth application in AAP? You put in a placeholder for the redirect which now has to be replaced by the actual value:

* Find the URL of the route `redhat-rhaap-portal`
* Append the string `/api/auth/rhaap/handler/frame`

NOTE:
====
For example, if the URL for the deployment is https://my-automation-portal-project.mycluster.com, then the Redirect URI value is https://my-automation-portal-project.mycluster.com/api/auth/rhaap/handler/frame
====

* Open the Ansible Automation Platform UI as user admin
* Go to **Access Management -> OAuth Applications**
* Click the OAuth application `selfservice`
* Replace the placeholder text in the Redirect URIs field with the value you determined from your OpenShift deployment.
* Click **Save**

And now it's finally time to log in to your **Self-Service Automation Portal**. In your browser, access the URL for the `redhat-rhaap-portal` deployment and log in with your AAP credentials for the admin user.

If everything was configured correctly you should be greeted with a nice new UI, showing one Job Template, the `Demo Job Template`. As a first test, execute the Template by clicking **Start**. After it has finished you can have a look at the logs for the steps executed. Have a look at the AAP Jobs view, the Template will show here as a Playbook run, too.

## Using the Self-Service Portal
















