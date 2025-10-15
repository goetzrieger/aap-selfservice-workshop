# Ansible Automation Platform (AAP) - Self Service Portal

This workshop will go through setting up AAP and the new AAP Self-Service Portal on Openshift. A well-equipped Single Node OpenShift Instance (e.g. 32 cores/128 GB RAM) should suffice.

## AAP Installation on OpenShift

Now we need to install Ansible Automation Platform 2.6, this is an easy task since it's done via an Operator:

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

== Installation of AAP Self-Service Portal

The Self-Service Portal is a customized version of Red Hat Developer Hub, which is based on the Backstage project. It includes a number of custom plugins to provide the features for automation self-service.
Currently it's only available on OpenShift, but a RHEL version is on the roadmap.

There are a good number of steps required to install it so let's go ahead. The full installation documentation can be found [here](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_self-service_automation_portal).

The Self-Service Portal is installed via a Helm Chart. The first step is to create an OAuth application in AAP for authentication:

* You should be logged i nto the AAP UI as user admin
* Navigate to **Access Management -> OAuth Applications**
* Click **Create OAuth Application**
* Complete the fields in the form:
 * **Name:** selfservice
 * **Organization:** default
 * **Authorization grant type:** Authorization code
 * **Client type:** Confidential
 * **Redirect URIs:** https//www.example.com, this will be replaced later






