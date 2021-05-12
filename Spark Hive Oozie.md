# Spark XML processing into Hive table through Oozie scheduling (as part of the BigInsights Suite)

* This document details the processing of XML files on HDFS into hive tables for further analysis. The XML files are of NMFP2 format and can be found at: https://searchwww.sec.gov/EDGARFSClient/jsp/EDGAR_MainAccess.jsp?search_text=*&sort=Date&formType=FormNMFP2&isAdv=true&stemming=true&numResults=10&numResults=10

* For each filing, you can click on the CIK number, then in the new window, scroll down looking for an NMFP2 filing on the left and click on the documents button for that filing. Then click on the primary_doc.xml link which contains the data for each filing. This process will use two of these XML files found with the EDGAR filings database. The specific XML files do not matter, but for this example CIK numbers 35315 and 277751 will be used. These files are included in the package. 

* The jar file (spark-xml_2.11-0.4.1.jar) responsible for ingesting the XMLs into spark is located at: 
https://mvnrepository.com/artifact/com.databricks/spark-xml_2.11/0.4.1

* Fully installed IOP package is required. 

## Procedure

* Download and extract sparkxml.tar.gz
Make a directory /sparkxml and un-tar the files in it
$ mkdir /ws
$ cd /ws
$ tar xvzf ~/Downloads/sparkxml.tar.gz

<p align="center">
<img src="https://github.com/chengp3/BigInsights-Articles/blob/master/images/sparkxmltar.png?raw=true">
 </p>

* Place the HDFS.sparkxml files into new folder on HDFS /ws/HDFS.sparkxml:
$ su - hdfs
$ hadoop fs -mkdir /ws
$ hadoop fs -put /ws/sparkxml/HDFS.sparkxml /ws/HDFS.sparkxml

* Confirm presence of files in Ambari Files View

<p align="center">
<img src="https://github.com/chengp3/BigInsights-Articles/blob/master/images/ambari.png?raw=true">
 </p>


It is important to put the XML files into the oozie folder as oozie will not recognize the input files unless they are together with the workflow.xml and script file (that we will define later). 

### Prepare the Hive db ahead of time by creating it in shell:

$ su - hive
$ hive 
hive> create database spark_test;
hive> show databases; 			----- to confirm its presence

* xml.scala 
This is the main file responsible for ingestion and entry of the XML files into Spark. Then it will digest it into a flattened data structure, create temporary tables, and finally enter these tables into the Hive db we created earlier. 
 
* sparkxml.sh
This starts the spark shell and runs our xml.scala script with the jar file we downloaded previously. 

* workflow.xml
This is a simple oozie workflow with a single action which runs sparkxml.sh.

* job.properties
Contains basic environment variable definitions required for oozie workflow to run, including where the data files and scripts reside on hdfs.

### Run the oozie workflow:
$ su - oozie
$ oozie job --oozie http://localhost:11000/oozie -config /ws/sparkxml/job.properties -run

Can track the job status by going to http://localhost:11000/oozie and hitting refresh until the job completes. 

<p align="center">
<img src="https://github.com/chengp3/BigInsights-Articles/blob/master/images/oozie.png?raw=true">
 </p>

### Confirm Hive table entry:
$ su - hive
$ hive
hive> use spark_test;
hive> show tables;
hive> select * from nfmp_generalinfo_t;
hive> select cik from nfmp_generalinfo_t;



## Troubleshooting:

On multi node clusters, please utilize impersonation to ensure oozie can run all commands necessary


