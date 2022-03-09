# Configure JBoss EAP cluster and AZURE_PING

In this documentation, we will setup a JBoss EAP cluster manually.

## Prerequites

We are using an Azure Marketplace offer to create VM resources, and it requires users to bring their own **RHSM credentials and Pool ID**.

## Create JBoss EAP VMs on Azure

We will using Red Hat JBoss Enterprise Application Platform (JBoss EAP) to create VM resources on Azure.

<details>
  <summary>
    <b>10min</b> Create RHEL VMs resources using the offer.
  </summary>

1. Access the offer from this link:[JBoss EAP on VMs](https://aka.ms/jboss-eap-on-vms). Or Visit the Azure portal, in the search box at the top of the page, type **Red Hat JBoss Enterprise Application Platform (JBoss EAP)**. When the suggestions start appearing, select the one and only match that appears in the **Marketplace** section.

1. Select **JBoss EAP 7 (BYOS) on RHEL 8 (PAYG) Clustered VM** from the **Plan** dropdown, then select **Create** to start.
1. In the **Basics** tab, create a new resource group called *jboss-demo-rg*.
1. Select *East US* as **Region**.
1. Type *jbossdemoVM* as **Virtual Machine name**.
1. Type *jbossdemoas* as **Availability set name**.
1. Type *azureadmin* as **Username**.
1. Leave **Authentication Type** to be *Password* as default.
1. Type *Password123456!* as **Password**, confirming it by re-enter it.
1. Select **Next: Virtual Machine Settings** to navigate to the next tab.
1. Type *3* as **Number of Instances**.
1. Leave other values as default, select **Next: JBoss EAP Settings** to navigate to the next tab.
1. Type *jbossadmin* as **JBoss EAP Admin username**.
1. Type *Secret123456* as **JBoss EAP password**.
1. Provide your RHSM credentials and Pool ID as instructed.
1. Select **Review + create**, then select **Create** to start the deployment.

After the deployment succeeded, you should be able to see three VMs, an Load balancer and two Storage accounts will be created.

</details>

## Verify the VM resources by creating an Application Gateway

We will create an Application Gateway to access the ports of the Load Balancer and the RHEL VMs. 

<details>
  <summary>
    <b>10min</b> Configure the Application Gateway
  </summary>

Please following [Quickstart: Direct web traffic with Azure Application Gateway - Azure portal](https://docs.microsoft.com/en-us/azure/application-gateway/quick-create-portal#create-an-application-gateway) to create the Azure Application Gateway, then return to this page.

Make sure you follow the tips mentioned in this documentation: [JBoss EAP on RHEL (clustered, multi-VM)](https://github.com/Azure/azure-quickstart-templates/tree/master/application-workloads/jboss/jboss-eap-clustered-multivm-rhel#validation-steps) at *Option 3 of 3. Using an Application Gateway*

Save the Public IP of the Application Gateway for later use.

</details>

## Setup public IP for RHEL Vms

To setup the cluster, we need to SSH to the VMs and modified the configure files. The first step is to assign public IPs for them.

<details>
  <summary>
    <b>5min</b> Configure public IP for created VMs
  </summary>

Follow this documentation and assign public IP to all the VMs created: [https://docs.microsoft.com/en-us/azure/virtual-network/ip-services/associate-public-ip-address-vm#azure-portal](https://docs.microsoft.com/en-us/azure/virtual-network/ip-services/associate-public-ip-address-vm#azure-portal), then go back to this documentation.

</details>

## Configure the cluster with Azure_PING

The Azure offer deployed three standalone nodes and configured them behind a Load Balancer to achieve HA.

We will form a cluster with three hosts: *jbossdemoVM0* will be the domain master, *jbossdemoVM1* and *jbossdemoVM2* will be the slaves.

### Stop the standalone service

Currently the VMs are running standalone server instances, we need to stop them before configuring cluster.

<details>
  <summary>
    <b>5min</b> Stop the standalone service running on VMs created by the offer.
  </summary>

1. SSH to *jbossdemoVM0* by running:

   ```bash
   ssh azureadmin@<jbossdemoVM0_Pub_IP>
   # Type Password123456! as password 
   ```

1. Stop the standalone server

   ```console
   cd /opt/rh/eap7/root/usr/share/wildfly/bin
   sudo ./jboss-cli.sh
    [disconnected /] connect
    [standalone@localhost:9990 /] shutdown
    [disconnected /] quit
   ```

1. Stop and disable the standalone daemon service

   ```bash
   sudo systemctl stop eap7-standalone.service
   sudo systemctl disable eap7-standalone.service
   ```

</details>

### Configure master host with AZURE_PING

We will configure *jbossdemoVM0* as the master host of the cluster.

<details>
  <summary>
    <b>10min</b> Configure the master host
  </summary>

1. SSH to *jbossdemoVM0* by running:

   ```bash
   ssh azureadmin@<jbossdemoVM0_Pub_IP>
   # Type Password123456! as password 
   ```

1. Create an user for slave hosts accessing master host:

   ```console
   cd /opt/rh/eap7/root/usr/share/wildfly/bin
   ./add-user.sh
   ```

   Follow the instructions and create a user *jbossmaster*/*Password~*.
   After confirming the user's management group, it will ask:

   ```console
   Is this new user going to be used for one AS process to connect to another AS process? 
   e.g. for a slave host controller connecting to the master or for a Remoting connection for server to server EJB calls.
   yes/no? 
   ```

   Answer yes, then it will print:

   ```console
   To represent the user add the following to the server-identities definition <secret value="cm17trb3MTA=" />
   ```

   Save the *secret value* for later use.
1. Modify host-master.xml

   ```bash
   cd /opt/rh/eap7/root/usr/share/wildfly/domain/configuration/
   sudo vi host-master.xml
   ```

   Find the *interfaces* configuration and update it to:

   ```xml
    <interfaces>
        <interface name="management">
           <inet-address value="${jboss.bind.address.management:<Master Host Private IP>}"/>
        </interface>
        <interface name="public">
           <inet-address value="${jboss.bind.address:<Master Host Private IP>}"/>
        </interface>    
        <interface name="unsecured">
           <inet-address value="<Master Host Private IP>" />    
        </interface>
    </interfaces> 
   ```

   The IP address *Master Host Private IP* here is the IP of the master host. You can find it through Azure portal by going to *jbossdemoVM0* -> *Networking* -> *NIC Private IP*.
1. Modify the domain.xml

   ```bash
   cd /opt/rh/eap7/root/usr/share/wildfly/domain/configuration/
   sudo vi domain.xml
   ```

   At the beginning of the file, name the domain by adding *name="domain1"* to *domain name="domain1" xmlns="urn:jboss:domain:16.0"*
   Find *subsystem xmlns="urn:jboss:domain:jgroups:8.0"* under profile *ha*, change the *stack name="udp"* to the following:

   ```xml
    <stack name="udp">
        <transport type="UDP" socket-binding="jgroups-udp">
            <property name="ip_mcast">
                false
            </property>
        </transport>
        <protocol type="azure.AZURE_PING">
            <property name="storage_account_name">
                <storage_account_name>
            </property>
            <property name="storage_access_key">
                <storage_access_key>
            </property>
            <property name="container">
                <container>
            </property>
        </protocol>
        <protocol type="MERGE3"/>
        <protocol type="FD_SOCK" socket-binding="jgroups-udp-fd"/>
        <protocol type="FD"/>
        <protocol type="VERIFY_SUSPECT"/>
        <protocol type="pbcast.NAKACK2">
            <property name="use_mcast_xmit">
                false
            </property>
            <property name="use_mcast_xmit_req">
                false
            </property>
        </protocol>
        <protocol type="UNICAST3"/>
        <protocol type="pbcast.STABLE"/>
        <protocol type="pbcast.GMS"/>
        <protocol type="UFC"/>
        <protocol type="FRAG2"/>
    </stack>
   ```

   The *storage_account_name*, *storage_access_key* and *container* can be found via Azure portal. In your resource group, find a Storage account named like *jbosstrgqnwtxzfa7hhbk* and select it. Select *Access keys* under *Security + networking*, then you can save the Storage account name, select *Show keys* then save the *key1* -> *Key* value. Select *Containers* under *Data storage*, there is a container named *eapblobcontainer*, save the name.

   Find *server-groups* and change the *profile* to *ha* and the *socket-binding-group ref* to *ha-sockets* for both *main-server-group*.
1. Start the master host

   ```bash
   cd /opt/rh/eap7/root/usr/share/wildfly/bin/
   sudo ./domain.sh --host-config=host-master.xml -Djboss.domain.base.dir=../domain
   ```

1. Verify the master host is started by going to *http://{Application Gatewa Public IP}:9990/console/index.html*. Type in *jbossadmin* and *Secret123456* as credentials. In *Runtime* -> *Topology*, you will see there is only one host called *master*, later we will configure slave hosts to join the cluster.

</details>

### Configure slave hosts with AZURE_PING

We will configure *jbossdemoVM1* and *jbossdemoVM2* as the slave hosts of the cluster.

<details>
  <summary>
    <b>10min</b> Configure the slave hosts
  </summary>

1. Modify host-slave.xml

   ```bash
   cd /opt/rh/eap7/root/usr/share/wildfly/domain/configuration/
   sudo vi host-slave.xml
   ```

   Find *security-realm name="ManagementRealm"*, update the *server-identities/secret value* to the printed secret value after you created the user by running *./add-user.sh*.
   Find *domain-controller* and update it to:

   ```xml
    <domain-controller>
        <remote security-realm="ManagementRealm" username="jbossmaster" >
            <discovery-options>
                <static-discovery name="primary" protocol="remote+http" host="<Master Host Private IP>" port="9990"/>
            </discovery-options>
        </remote>
    </domain-controller>
   ```

   Find *interfaces* and update it to:

   ```xml
    <interfaces>
        <interface name="management">
            <inet-address value="${jboss.bind.address.management:<Slave Host Private IP>}"/>
        </interface>
        <interface name="public">
            <inet-address value="${jboss.bind.address:<Slave Host Private IP>}"/>
        </interface>
        <interface name="unsecured">
            <inet-address value="<Slave Host Private IP>" />
        </interface>
    </interfaces>
   ```

   The *Slave Host Private IP* can be found the same way you find the *Master Host Private IP*.

   Find *servers* and update it to:

   ```xml
   <servers>
        <server name="server-one" group="main-server-group"/>
        <server name="server-two" group="main-server-group">
            <socket-bindings port-offset="150"/>
        </server>
    </servers>
   ```

   Make sure you are using unique names for servers defined in *jbossdemoVM1* and *jbossdemoVM2*.
1. Update domain.xml
   Update the domain.xml as we did for host master.
1. Start the slave host

   ```bash
   cd /opt/rh/eap7/root/usr/share/wildfly/bin/
   sudo ./domain.sh --host-config=host-slave.xml
   ```

1. Verify the master host is started by going to *http://{Application Gatewa Public IP}:9990/console/index.html*. Type in *jbossadmin* and *Secret123456* as credentials. In *Runtime* -> *Topology*, you will see there is only one host called *master*, later we will configure slave hosts to join the cluster.

</details>

### Verify that AZURE_PING works as expected

After the cluster is configured, we need to deploy a sample application to verify the AZURE_PING is working as expected.

<details>
  <summary>
    <b>5min</b> Verify the AZURE_PING works by deploying sample application
  </summary>

1. Checkout [cluster-demo](https://github.com/liweinan/cluster-demo)
1. Run `mvn clean install`
1. Follow [CHAPTER 7. DEPLOYING APPLICATIONS](https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/7.0/html/configuration_guide/deploying_applications#deploy_app_domain_console) and deploy the application.
1. Go to the Storage account -> container *eapblobcontainer*, there should be a file named like *ejb-bbee39df-e0cb-fa7e-d30a-19190f68f042.jbossdemovm1-server-one.list*. Download and open it you should see:

   ```console
    jbossdemovm2:server-six 	974af960-be01-38a9-7e4f-544e3c43051b 	127.0.0.1:55350 	F
    jbossdemovm2:server-five 	2038f710-5cfc-4157-61e7-59cb7988f7ca 	127.0.0.1:55200 	T
   ```

   It means the *jbossdemovm2:server-five* is the coordinator, new members added to the group will load this *.list* file and learn about all the members.

</details>

## Clean up the resources

To avoid Azure charges, you should clean up unnecessary resources.

<details>
  <summary>
    <b>5min</b> Delete Azure resources
  </summary>

When the cluster is no longer needed, use the [az group delete](https://docs.microsoft.com/en-us/cli/azure/group#az_group_delete) command to remove the resource group and all related resources.

```azurecli
az group delete --name $RESOURCE_GROUP_NAME --yes --no-wait
```

</details>
