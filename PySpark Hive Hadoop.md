# PySpark

* Quick introduction
* Steps to configure PySpark
* Example operations

## Introduction: 
Spark is a type of interface for running calculations on big data. As the amount of information produced and/or collected continues to expand astronomically, the methods of handling this data continues to evolve. Hadoop is currently a workhorse, allowing economical scaling of hardware to meet the needs of data analysis. Spark continues on this theme by working to streamline data flow through memory if possible, and combining certain processes. The end result is a 100x speed increase over Hadoop in some situations. Spark is also considered more user friendly, and is easier to manipulate with higher level languages, including Python, Java and Scala. We will investigate a basic use case for PySpark, the Python implementation of Spark. 

#### How to install PySpark on CentOS:

Spark is installed as part of the IBM open platform package (this document will make use of IOP 4.1.0.0). The version included in IOP is Solr 2.1.0. 

The python package used here is 2.7.5. We will be working within jupyter notebook, a standard python development environment. 

We will use a virtual environment to maintain version control of our packages. Install with:
```
$ pip install virtualenv
```

Once virtual env is installed, you can create a virtual env called “test-env” with:
```
$ virtualenv test-env 
```

This will create a new directory in your current working directory. You can activate the environment with:
```
$ . bin/activate
```

Once activated, you can install ipython and then jupyter (the notebook environment with:
```
$ pip install ipython
$ pip install jupyter
```

We have to create a settings profile with:
```
$ jupyter notebook --generate-config
```

Go into the directory created, and edit the file ipython_notebook_config.py. Change the c.NotebookApp.ip setting from ‘localhost’ to ‘127.0.0.1’, and uncomment the line. 

Now run jupyter notebook with pyspark environment:
```
$ PYSPARK_DRIVER_PYTHON=jupyter PYSPARK_DRIVER_PYTHON_OPTS=”notebook” spark-2.1.0-bin-hadoop2.7/bin/pyspark
```

A browser window will open with cells awaiting python code to be inputted.

##### Spark

PySpark runs using a python object called an “RDD”, which stands for “resilient distributed dataset”. The RDD contains data like other python objects, but also contains functions to  handle the fault tolerance that is inherent to distributed computing. We will see some of these properties soon. For now we can create and manipulate a simple dataset:

```
from itertools import cycle
n = zip(cycle([“even”,”odd”]),range(20))
numbers = sc.parallelize(n)
increased = numbers.mapValues(lambda x:x+1)
grouped = increased.reduceByKey(add)
grouped.collect()
```

Now if we run grouped.toDebugString(), we can see the chain of interactions used to produce the most current version of our RDD. This provides fault tolerance by providing the ability to retrace steps in the event of failure by any of the executors.  

```
grouped.toDebugString()
```

There are of course many more operations available in PySpark and only a few properties were demonstrated here.


#### Conclusion:

PySpark is a scalable, fault-tolerant, and convenient tool for distributed computing and big data analysis.



# Hive

Hive is a system that operates on unstructured data stored in HDFS. Hive commands are SQL formatted, and these SQL commands are processed within the Hive engine to produce MapReduce scripts. 

Hive is able to manipulate XML files using a plugin called serializer-deserializer (“SerDe”). These are steps to configure the hive process to implement this plugin. 

Configuring Hive for XML file processing:

These steps are performed with installation of Ambari 4.1.0.0. Hive is installed as part of this package. The XML SerDe .jar file is located at https://github.com/dvasilen/Hive-XML-SerDe/wiki/XML-data-sources

Download the .jar file and extract to /lib/. Once downloaded, start a hive session by changing directory to the hive initialization script, execute it, and engage the .jar plugin (replace the x’s with appropriate version number):
```
$ cd /usr/iop/current/hive-client/bin
$ hive
hive> hive --auxpath /lib/hivexmlserde-x.x.x.x.jar
```

Once executed, hive is ready to serialize and deserialize XML files. We can use a sample XML file that contains ebay transaction data downloaded here (http://www.cs.washington.edu/research/xmldatasets/data/auctions/ebay.xml.gz)

Then we may create a table and process the data:
```
CREATE TABLE ebay_listing(seller_name STRING,
seller_rating BIGINT, bidder_name STRING,
location STRING, bid_history map<string,string>,
item_info map<string,string>)
ROW FORMAT SERDE 'com.ibm.spss.hive.serde2.xml.XmlSerDe'
WITH SERDEPROPERTIES (
"column.xpath.seller_name"="/listing/seller_info/seller_name/text()",
"column.xpath.seller_rating"="/listing/seller_info/seller_rating/text()",
"column.xpath.bidder_name"="/listing/auction_info/high_bidder/bidder_name/text()",
"column.xpath.location"="/listing/auction_info/location/text()",
"column.xpath.bid_history"="/listing/bid_history/*",
"column.xpath.item_info"="/listing/item_info/*"
)
STORED AS
INPUTFORMAT 'com.ibm.spss.hive.serde2.xml.XmlInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'
TBLPROPERTIES (
"xmlinput.start"="<listing>",
"xmlinput.end"="</listing>"
);
```

Once the table is created, load the xml data with these commands (replacing /path/ with the path to your data folder: 
```
LOAD DATA LOCAL INPATH '/path/ebay.xml'
OVERWRITE INTO TABLE ebay_listing;
```

Querying is then simple and similar to SQL:
```
SELECT seller_name, bidder_name, location, bid_history["highest_bid_amount"], item_info["cpu"]
FROM ebay_listing LIMIT 1;
```

# Hadoop security: 
Hadoop is the de facto tool used for big data storage and retrieval. One shortcoming in a default hadoop file system is the lack of security regarding its files. For example, a non-authorized user may access blocks in the HDFS locally, read, or even manipulate them. This is obviously not ideal for sensitive data or even data integrity. 

To address this issue, data must be secured at “rest”, meaning that this access is not allowed. The security component to configure is called Key Management Service (KMS), which stores encryption keys. To set up KMS, run the following commands: 

```
mkdir -p /usr/TDE_demo
$ cp /usr/iop/current/hadoop/mapreduce.tar.gz /usr/TDE_demo/
$ export TDE_ROOT=/usr/TDE_demo
$ cd /usr/TDE_demo
$ tar –xvzf mapreduce.tar.gz
$ cd hadoop/sbin/
$ ./kms.sh run
```

In Ambari, configure the HDFS by clicking HDFS, then Configs, then Advanced. Add the following key and value to custom core-site, replacing “mymachine.domain.com” with your hostname:
hadoop.security.key.provider.path = kms://http@mymachine.domain.com:16000/kms

Then add the following key and value to custom hdfs-site, again replacing “mymachine.domain.com” with the hostname:
dfs.encryption.key.provider.uri = kms://http@ mymachine.domain.com:16000/kms

Restart HDFS, MapReduce2, and YARN.

Create an encryption key named “key_demo” with: 
```
$ su hdfs
$ hadoop key create key_demo -size 256
$ hadoop key list -metadata
```

Create an encryption zone
```
$ hdfs dfs -mkdir /encryption_zone
$ hdfs crypto -createZone -keyName key_demo -path /encryption_zone
$ hdfs crypto -listZones
```

Users who do not have permission from now on (even root), may not access files in the encryption zone.
