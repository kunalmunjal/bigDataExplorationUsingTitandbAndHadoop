## Introduction
We begin with an introduction to the Tech Stack that we used before briefly elucidating the schema of the dataset that was ingested. Onwards, we elaborate the Setup, and then the Data Ingestion methodology including reasons for choosing different parts of the stack.

The second part deals with Indexing and Retrieval of data using Gremlin from TitanDB. We conclude with a short summary and an attached appendix.

## Tech Stack
![tech-stack](https://github.com/jayantsharma/bigDataExplorationUsingTitandbAndHadoop/blob/master/images/tech_stack.png)

### Details
1. _Spark_ in conjunction with _HDFS_ for pre-processing of data.
1. _TitanDB_ configured with _Cassandra_ as the storage-backend and _ElasticSearch_ as the indexing-backend serves as our graph database.
1. Spark interacting with HDFS is responsible for _Bulk Ingestion_ of data in TitanDB.
1. _MapReduce_ leveraged to build indexes over the graph database.

## Schema
In contrast to most mainstream technologies, TitanDB is not _schemaless_. Vertex and Edge labels as well as properties and their datatypes need to be declared before starting to use the database. This is generally done by a call to the graph's _ManagementSystem_ (for eg, refer: scripts/load-users.groovy).

![schema](https://github.com/jayantsharma/bigDataExplorationUsingTitandbAndHadoop/blob/master/images/schema.png)

The StackOverflow dataset, as noted in the proposal consists of 3 main elements:
1. Users - The Stack Overflow users
2. Posts - Questions or Answers
3. Comments - User comments on posts

Let's run through the sample schema pictured above.
1. John, Bill and Silly are 3 fictional users.
2. John asks the question "How to compute sum of ..." to which Bill answers "(1 to N toList) ...". Note that both the question and answer are posts, but in addition, the _answer\_post_ has an _answerTo_ relationship with the _question\_post_.

## SETUP (TitanDB - Cassandra - Elasticsearch)

__NOTE__: All versions of the following support software have been selected because of limitations with TitanDB 1.0.0. The last public release of TitanDB was over 2 years ago, when they only supported Hadoop 1.

### Elasticsearch
1. Get [Elasticsearch 1.5.2](https://github.com/elastic/elasticsearch/releases/tag/v1.5.2) from the _releases_ section of the Github repo.
2. Use [Maven](http://maven.apache.org) to build elasticsearch from the sourcecode: `mvn clean package -DskipTests`.
3. Unzip the built package, and execute `bin/elasticsearch`. Verify elastic is up using: `curl -X GET http://localhost:9200/`.

### Cassandra
2. Get [Cassandra 2.1.19](http://www.apache.org/dyn/closer.lua/cassandra/2.1.19/apache-cassandra-2.1.19-bin.tar.gz) from the  _Downloads_ section of their website.
2. Execute `bin/cassandra` from the root directory of the un-archived package. The default settings worked for me with storage set to PROJECT\_ROOT/data/data. You might have to play with the JAVA\_HOME variable to get this right though.
2. You can start cassandra in the background with `cassandra -f` and forget about it.

### Spark
Spark is required because we need to process large XML files. This might sound like a snazzy decision, but the mature Spark XML library which creates Dataframes out of XML data was a no brainer.
1. Get [Spark 1.6.1](https://d3kbcqa49mib13.cloudfront.net/spark-1.6.1-bin-hadoop1.tgz).
2. Edit _conf/spark-env.sh_ to set the HADOOP\_CONF\_DIR environment variable to point to the _conf_ directory of your Hadoop installation, so that it integrates with HDFS. Now by default, File I/O happens with HDFS.
3. Add the following line to `conf/spark-defaults.conf`, so that the necessary libraries are loaded each time Spark is started.
```
spark.jars.packages   com.databricks:spark-xml_2.10:0.3.5,org.apache.hadoop:hadoop-client:1.2.1,com.databricks:spark-csv_2.10:1.5.0
```

### Hadoop/HDFS
Not an essential per se, but the Big Data requirement(parsing of XML files running into GBs with Spark) alongwith the way TitanDB's bulk loading program works(only reads from HDFS) requires this.
1. Get your preferred format from Apache Archives of [Hadoop 1.2.1](https://archive.apache.org/dist/hadoop/common/hadoop-1.2.1/).
2. Conf files from in the `conf/hadoop` directory(in repo) to be used. Specifically, `dfs.data.dir` and `dfs.name.dir` need to be set to some location in the local filesystem APART FROM /tmp, since /tmp is cleared every time the machine powers down.

### Titan
3. Get Titan 1.0.0 from the [Downloads](https://github.com/thinkaurelius/titan/wiki/Downloads) page @ Titan.
4. Edit `bin/gremlin.sh` to ensure CLASSPATH includes the path to Hadoop's conf directory. Refer `bin/gremlin.sh`.
4. Copy the opencsv and groovycsv jars from the **lib** directory of the repo to lib directory of your TitanDB project dir.
4. `bin/gremlin.sh` from inside the project directory, and you're good to go.


## Data Ingestion

### General Strategy
1. Read XML files from HDFS using Spark, take attributes of interest and write to CSV again in HDFS.
2. Use BulkLoaderVertexProgram(running over Spark) to digest the resulting CSV file and add nodes to graph.

### Detailed Example for User nodes
#### Convert Data to CSV using Spark
##### Load Data using spark-xml
- Modify XML in-place using `sed` so it's readable. This is a limitation of Spark-XML, it can't read self-closing XML tags.
```sh
sed -i -e 's/\/>/><foo>bar<\/foo><\/row>/' Users.xml
```
##### Move data to HDFS
```sh
# Create input directory if it doesn't exist
hadoop fs -ls
hadoop fs -mkdir input

# Move Users.xml to input
# moveFromLocal instead of copyFromLocal if you're short on space
hadoop fs -copyFromLocal Users.xml input/
```
##### Process XML in Spark
The following lines of codes are most conveniently implemented on Spark Shell (`bin/spark-shell`).
```scala
// In order to read XML into Spark Dataframes, Spark SQL is required.
import org.apache.spark.sql.SQLContext
val sqlContext = new SQLContext(sc)

// Read the <row> tags of the XML file that is loaded from the HDFS location "input/Users.xml". The <row> tags could more appropriately have been <user> tags, but that's how the file is formatted.
// Play with the Dataframe using df.show() or df.filter functions to get a sense of what the data looks like.
val df = sqlContext.read.format("com.databricks.spark.xml").option("rowTag", "row").load("input/Users.xml")

// Select the attributes of interest, write them using CSV format with no header and NULLs formatted as empty strings and write to graph/nodes/users directory in HDFS.
// The directory MUST be empty if it exists. If it doesn't exist, Spark will create it.
df.select("@Id", "@DisplayName", "@Age", "@Location", "@UpVotes", "@DownVotes", "@Reputation").write.format("com.databricks.spark.csv").option("nullValue", "").save("output/users")
```

#### Ingest Data in TitanDB
We are going to leverage Gremlin-Hadoop (formerly TinkerPop Furnace) to bulk-process our data. Gremlin-Hadoop allows bulk ingestion of data using its BulkLoaderVertexProgram via reading of standard as well as arbitrarily formatted files (using Hadoop's ScriptInputFormat). CSV is a custom or arbitrary format because the standard formats for graph ingestion are GraphML and GraphSON, based on XML and JSON respectively. The formats for both of these are fairly sophisticated and the general method of producing such files is generally by outputting a graph in one of these formats, which doesn't work for our case.

An obvious implication of the custom format is a script that can parse the format. Therefore, there are 3 files that we need to put in place for TitanDB to process the data.
1. *scripts/load\_users.groovy* : Creates the attributes and label of nodes (here, users) that we need.
2. *conf/hadoop-graph/users-hadoop-load.properties* : Specifies location of input data and the parsing script.
3. *scripts/users-script-input.groovy* : The parsing script itself, this needs to be in HDFS too.

Once these are in-place, just execute the *load\_users.groovy* script on the Gremlin shell:
```groovy
:load scripts/load_users.groovy
```

### Loading Edges
The process for loading edges varies because of a [long-standing bug](https://issues.apache.org/jira/browse/TINKERPOP-432) in the version of TinkerPOP that TitanDB uses. Instead of using BLVP, edges are therefore loaded transactionally. The below example creates directed edges with the label _createdBy_ from posts to users, signifying the relation that a given post was created by the connected user.
#### Process using Spark to get partial adjacency list (only out-edges, no in-edges)
```scala
// Get Dataframe as earlier, then continue using below
// Take a fraction of data points, if you so wish
var filtered_posts_users = df.filter(df.col("@OwnerUserId").isNotNull).select("@Id", "@OwnerUserId").limit(100000)
// write to CSV
filtered_posts_users.write.format("com.databricks.spark.csv").option("nullValue", "").save("graph/edges/posts_owners")

// PLS IGNORE
// filtered_posts_users = filtered_posts_users.limit(filtered_posts_users.count() / 10)
// val users_then_posts = filtered_posts_users.withColumnRenamed("@OwnerUserId", "a").withColumn("b", lit(null: String)).withColumnRenamed("@Id", "c").select("a","b","c")
// posts_then_users.unionAll(users_then_posts).write.format("com.databricks.spark.csv").option("nullValue", "").save("graph/edges/posts_owners")
```
#### Ingesting in TitanDB
We copy the data from HDFS to the local filesystem and use just one script: *scripts/load_user_posts_edges.groovy* to load the edges into the graph.

## Sample Queries
```groovy
// Get users with 'murakami' in their name(text-insensitive) with a reputation between 10000 and 50000
// Can be used to get the scala questions asked by high-rep users by leveraging wildcard search on tags of posts and range search on user-rep
g.V().has("DisplayName", textContainsRegex("murakami")).has("Reputation", gt(1000)).count()

// Get users connected to a certain user via answers to their asked questions (includes duplicates)
g.V().has('bulkLoader.vertex.id', 'user:91').in('createdBy').has('PostTypeId', 1).in('answerTo').out('createdBy').values('DisplayName').dedup()

// Get users connected to a given user u1 via comments on questions asked by u1
g.V().has('bulkLoader.vertex.id', 'user:91').in('createdBy').has("PostTypeId", 1).in('commentOn').out('createdBy').values("DisplayName").dedup()
```

## Sample Queries Part 2
```groovy
// Get Users with names either 'John' or 'Doe' and sort and find the top 5 according to their reputations and find out the questions they have asked having tags 'Java' for example. This includes duplicates.
g.V().has("DisplayName", textContainsRegex("John","Doe")).order().by('Reputation',decr).limit(5).in("createdBy").has("PostTypeId",1).has("Tags", textContainsRegex("Java"))
// Get Users with name 'John' and find out the answers they have written containing the tag 'Java'
g.V().has("DisplayName", textContainsRegex("John")).in("createdBy").has("PostTypeId",2).has("Tags=", textContainsRegex("Java"))

## Indexing
### Composite vs Mixed Indexes
### Configuring app to never do full graph scans (force-index)
### Build Index
```groovy
graph = TitanFactory.open('conf/se_dump.properties')
mgmt = graph.openManagement()
name = mgmt.getPropertyKey("DisplayName")
/*
 Build a graph-centric index on all the users indexing them by their name.
*/
mgmt.buildIndex('usersByName', Vertex.class).addKey(name, Mapping.TEXT.asParameter()).indexOnly("user").buildMixedIndex("search")
```

### Enable this:
* g.V().has('name', textContains('hercules')).has('age', inside(20, 50)).order().by('age', decr).limit(10) 
