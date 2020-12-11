# HDP-Security-Practice-Test
## Step 1 :- Setup HDP 2.6.5 Cluster with Ambaro 2.6.2
### Login to Ambari
Login to Ambari web UI by opening http://AMBARI_PUBLIC_IP:8080 and log in with admin/admin


You will see a list of Hadoop components running on your cluster on the left side of the page

They should all show green (ie started) status. If not, start them by Ambari via 'Service Actions' menu for that service

#### Question :- From Ambari how do you check the cluster name?
#### How will you change the password for the logged in User?
#### Please install Hive Service. 

Import sample data into Hive
Run below on the node where HiveServer2 is installed to download data and import it into a Hive table for later labs

You can either find the node using Ambari as outlined in Lab 1
Download and import data
    
    cd /tmp
    wget https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/labdata/sample_07.csv
    wget https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/labdata/sample_08.csv
    
Create user dir for admin, sales1 and hr1
     
     sudo -u hdfs hdfs dfs  -mkdir /user/admin
     sudo -u hdfs hdfs dfs  -chown admin:hadoop /user/admin

     sudo -u hdfs hdfs dfs  -mkdir /user/sales1
     sudo -u hdfs hdfs dfs  -chown sales1:hadoop /user/sales1

     sudo -u hdfs hdfs dfs  -mkdir /user/hr1
     sudo -u hdfs hdfs dfs  -chown hr1:hadoop /user/hr1   
     
     
Now create Hive table in default database by
Start beeline shell from the node where Hive is installed:

     beeline -n admin -u "jdbc:hive2://localhost:10000/default"

At beeline prompt, run below:

       CREATE TABLE `sample_07` (
      `code` string ,
      `description` string ,  
      `total_emp` int ,  
      `salary` int )
      ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TextFile;
      load data local inpath '/tmp/sample_07.csv' into table sample_07;
      CREATE TABLE `sample_08` (
      `code` string ,
      `description` string ,  
      `total_emp` int ,  
      `salary` int )
      ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TextFile;
      load data local inpath '/tmp/sample_08.csv' into table sample_08;
Notice that in the JDBC connect string for connecting to an unsecured Hive while its running in default (ie binary) transport mode :

port is 10000
no kerberos principal was needed
This will change after we:

#### enable kerberos
configure Hive for http transport mode (to go through Knox)


#### Why is security needed?
HDFS access on unsecured cluster
On your unsecured cluster try to access a restricted dir in HDFS
      
      hdfs dfs -ls /tmp/hive   

#### this should fail with Permission Denied
Now try again after setting HADOOP_USER_NAME env var

      export HADOOP_USER_NAME=hdfs
      hdfs dfs -ls /tmp/hive   
      
#### this shows the file listing!
Unset the env var and it will fail again
unset HADOOP_USER_NAME
hdfs dfs -ls /tmp/hive  


#### WebHDFS access on unsecured cluster
From node running NameNode, make a WebHDFS request using below command:

    curl -sk -L "http://$(hostname -f):50070/webhdfs/v1/user/?op=LISTSTATUS"

In the absence of Knox, notice it goes over HTTP (not HTTPS) on port 50070 and no credentials were needed
Web UI access on unsecured cluster
From Ambari notice you can open the WebUIs without any authentication

HDFS > Quicklinks > NameNode UI
Mapreduce > Quicklinks > JobHistory UI
YARN > Quicklinks > ResourceManager UI
This should tell you why kerberos (and other security) is needed on Hadoop :)

## Install Additional Components
### Install Knox via Ambari
Login to Ambari web UI by opening http://AMBARI_PUBLIC_IP:8080 and log in with admin/admin

Use the 'Add Service' Wizard (under 'Actions' dropdown, near bottom left of page) to install Knox on a node other than the one running Ambari

Make sure not to install Knox on same node as Ambari (or if you must, change its port from 8443)
Reason: in a later lab after we enable SSL for Ambari, it will run on port 8443
When prompted for the Knox Master Secret, set it to knox
Do not use password with special characters (like #, $ etc) here as seems beeline may have problem with it


## Lab 2
### Review use case
Use case: Customer has an existing cluster which they would like you to secure for them

Current setup:

The customer has multiple organizational groups (i.e. sales, hr, legal) which contain business users (sales1, hr1, legal1 etc) and peter

      ldapsearch -h sme-2012-ad.support.com -b "DC=support,DC=com" -D "test1@support.com" -w (sAMAccountname=peter)"

Hadoop cluster running HDP has already been setup using Ambari (including HDFS, YARN, Hive, Hbase, Solr, Zookeeper)
Goals:

#### Integrate Ambari with AD - so that peter can administer the cluster
      Configure Ambari for LDAP Authentication 

Integrate Hadoop nodes OS with AD - so business users are recognized and can submit Hadoop jobs



Enable kerberos using KDC - to secured the cluster and enable authentication
Install Ranger and enable Hadoop plugins - to allow admin to setup authorization policies and review audits across Hadoop components
Install Ranger KMS and enable HDFS encryption - to be able to create encryption zones
Encrypt Hive backing dirs - to protect hive tables
Configure Ranger policies to:
Protect /sales HDFS dir - so only sales group has access to it
Protect sales hive table - so only sales group has access to it
Protect sales HBase table - so only sales group has access to it
Install Knox and integrate with AD - for perimeter security and give clients access to APIs w/o dealing with kerberos
Enable Ambari views to work on secured cluster
We will run through a series of labs and step by step, achieve all of the above goals




