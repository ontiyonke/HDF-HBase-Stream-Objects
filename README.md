# Introduction:

Often times there will be a need to ingest binary files to Hadoop (like PDF, JPG, PNG) where you will want to store them in HBase directly and not on HDFS itself. This article describes an example how this could be achieved.


The maximum number of files in HDFS depends on the amount of memory available for the NameNode.

Each file object and each block object takes about 150 bytes of the memory. For example, if you have 10 million files and each file has 1 one block each, then you would need about 3GB of memory for the NameNode. This could pose a problem if you would like to store trillions of files, where you will run out of RAM on the NameNode trying to track all of these files.

One option to resolve this issue is to store your blobs in an Object store. There are many to choose from, especially if you use your favorite cloud provider. Another alternative would be to look at Ozone: https://hortonworks.com/blog/ozone-object-store-hdfs/

Below is an example how you can use HDF to ingest these Blobs into HBase directly, as another field. HBase has support for MOB, Medium Objects, which is a way to store objects of around 10MB or so in HBase directly. The article describes MOB in more detail: https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.5/bk_data-access/content/ch_MOB-support.html

Hortonworks Dataflow (HDF), can be used to visually design a flow whereby you can ingest from a directly continuously, add extra fields, compress, encrypt, hash and then store this data in HBase. Any application can then connect to HBase and retrieve these objects at high speed. For example, a document system, images for a website's catalog etc.

Here is a high level overview of the flow (also attached as a template file at the end of this article):
![alt text](https://github.com/willie-engelbrecht/HDF-HBase-Stream-Objects/blob/master/HDF-HBase-1.jpg "HDF Flow Design into HBase")


Let's break this down, each step:
* ListFile: Watches a configured directory, and generates a list of filenames when new files arrives
* FetchFile: Uses the list generated from the first processor, and reads those files from disk and streams into HDF
* HashFile: (Optional step) Hash the contents of the file, with md5, sha1, sha2 etc
* UpdateAttribute: (Optional step) Add additional attributes to the file read, for example author name, date ingested etc
* CompressContent: Compresses the file, using bzip, gz, snappy etc
* Base64EncodeContent: Changes the binary data to base64 representation for easier storage in HBase
* AttributesToJSON: Convert all attributes of the FlowFile (like filename, date, extra attributes etc) as as JSON file
* PutHBaseJSON: Take the JSON from the previous step, and store as key=>value in a column family

Also, one last processor which splits out from Base64EncodeContent to PutHBaseCell, which stores the actual file/object in HBase, also part of the column family.

As an example, here is the output you can expect from a sample PDF file stored in HBase:
![alt text](https://github.com/willie-engelbrecht/HDF-HBase-Stream-Objects/blob/master/HDF-HBase-5.JPG "Viewing HBase")


For the HBase processors, you will need to configure a controller service to define where your Zookeeper is in order to find your HBase servers.

PutHBaseCell:
![alt text](https://github.com/willie-engelbrecht/HDF-HBase-Stream-Objects/blob/master/HDF-HBase-3.jpg "Configuring PutHBaseCell")


Which in turn points to the controller service (HBase_1_1_2_ClientService):
![alt text](https://github.com/willie-engelbrecht/HDF-HBase-Stream-Objects/blob/master/HDF-HBase-2.jpg "Configuring HBase Controller Service")


Additionally, here is an example to read the same objects from HBase, and store them back to the file system:
![alt text](https://github.com/willie-engelbrecht/HDF-HBase-Stream-Objects/blob/master/HDF-HBase-4.jpg "Reading back from HBase")


As you can see, it's pretty much the reverse when writing to HBase initially:
* Read from HBase
* Extract the original filename from the HBase read
* Decode from Base64 back to binary
* Decompress
* Write to disk, with the original filename

Have a look at the attached hbasewriteexample.xml template, which you can import into your HDF environment to play with.
