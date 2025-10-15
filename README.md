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
* Replace the **spec** section with this content:

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



