Deploying a shared production ready MongoDB cluster with Ansible
------------------------------------------------------------------------------

In this example we demonstrate how we can orchestrate the deployment of a production grade MongoDB Cluster. The functionality of this example includes:

1) Deploying a N node MongoDB cluster, which has N shards and N replication nodes.

2) Scale out capability. Expand the Cluster by adding nodes to the cluster.

3) Security, All the mongodb process are secured using the best practices.

###Deployment Architecture.

To better explain the deployment architecture let's take an example where we are deploying a 3 node MongoDB cluster ( Minimum recommended by MongoDB).

The way Ansible configures the three nodes is as follows:

1) Install the mongodb software on all nodes.

2) Creates 3 replication sets, with one primary on each node and the rest two acting as secondaries.

3) Configures MongodDB configuration DB servers as listed in the inventory section[mongocservers]. Recommended number is 3, so it can be the same three servers as the datanodes.

4) Configures a Mongos server as listed in the inventory file [mongosservers].

5) Adds 3 Shards each belonging to individual replication sets.

6) All the processes, mongod,mogos are secured using the keyfiles.

Once the cluster is deployed, if we want to scale the cluster, Ansible configures it as follows:

1) Install the MongoDB application on the new node.

2) Configure the replication set with primary as the new node and the secondaries as listed in the inventory file [replicationservers]. ( don't forget to add the new node also in the replicationservers section]

3) Adds a new shard to the mongos service pointing to the new replication set.


#### Pre-requisite 

1) Update the group_vars/all file which contains site specific parmaters, especially the section which contains the mapping of the hostname's and the ports that it should use for the mongod process. Please do make sure the ansible hostname matches the same. Also dont forget to add the variable when adding a new node.

2) The default directory for storing data is /data, please do change it if requried, also make sure it has sufficient space 10G recommended.

###The following example deploys a three node MongoDB Cluster

The inventory file looks as follows:

		#The site wide list of mongodb servers
		[mongoservers]
		mongo1
		mongo2
		mongo3

		#The list of servers where replication should happen, including the master server.
		[replicationservers]
		mongo1
		mongo2
		mongo3

		#The list of mongodb configuration servers, make sure it is 1 or 3
		[mongocservers]
		mongo1
		mongo2
		mongo3

		#The list of servers where mongos servers would run. 
		[mongosservers]
		mongos

Build the site with the following command:

		ansible-playbook -i hosts site.yml

###Verifying the deployed MongoDB Cluster

Once completed we can check replication set availibitly by connecting to individual primary replication set nodes, 'mongo --host <ip host> --port <port number> 
and issue the command to query the status of replication set, we should get a similar output.

		
		web2:PRIMARY> rs.status()
		{
			"set" : "web2",
			"date" : ISODate("2013-03-19T10:26:35Z"),
			"myState" : 1,
			"members" : [
			{
				"_id" : 0,
				"name" : "web2:2013",
				"health" : 1,
				"state" : 1,
				"stateStr" : "PRIMARY",
				"uptime" : 102,
				"optime" : Timestamp(1363688755000, 1),
				"optimeDate" : ISODate("2013-03-19T10:25:55Z"),
				"self" : true
			},
		{
				"_id" : 1,
				"name" : "web3:2013",
				"health" : 1,
				"state" : 2,
				"stateStr" : "SECONDARY",
				"uptime" : 40,
				"optime" : Timestamp(1363688755000, 1),
				"optimeDate" : ISODate("2013-03-19T10:25:55Z"),
				"lastHeartbeat" : ISODate("2013-03-19T10:26:33Z"),
				"pingMs" : 1
			}
		],
		"ok" : 1
		}


we can check the status of the Shards as follows: connect to the mongos service 'mongos --host <ip of mongos server> --port 8888'
and issue the following command to get the status of the Shards.


		 
		mongos> sh.status()
		--- Sharding Status --- 
		  sharding version: { "_id" : 1, "version" : 3 }
		  shards:
			{  "_id" : "web2",  "host" : "web2/web2:2013,web3:2013" }
			{  "_id" : "web3",  "host" : "web3/web2:2014,web3:2014" }
  		databases:
			{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }


###We can also make sure the Sharding works by creating a database,collection and populate it with documents and check if the chunks of the collection are balanced equally across nodes.

The above mentioned steps can be tested with an automated playbook.

Issue the following command to run the test. In variable passed make sure the servername is one of any mongos server.
		
		ansible-playbook -i hosts playbooks/testsharding.yml -e servername=mongos


Once the playbook completes, we check if the shadring has succeded by logging on to any mongos server and issuing the following command. The output display the number of chunks spread across the shards.

		mongos> sh.status()
			--- Sharding Status --- 
  			sharding version: { "_id" : 1, "version" : 3 }
  			shards:
			{  "_id" : "bensible",  "host" : "bensible/bensible:20103,web2:20103,web3:20103" }
			{  "_id" : "web2",  "host" : "web2/bensible:20105,web2:20105,web3:20105" }
			{  "_id" : "web3",  "host" : "web3/bensible:20102,web2:20102,web3:20102" }
  			databases:
			{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
			{  "_id" : "test",  "partitioned" : true,  "primary" : "web3" }
			
				test.test_collection chunks:
				
				bensible	7
				web2	6
				web3	7
			
			

 
### Adding a new node to the Cluster

To add a new node to the configured MongoDb Cluster, setup the inventory file as follows:

		#The site wide list of mongodb servers
		[mongoservers]
		mongo1
		mongo2
		mongo3
		mongo4

		#The list of servers where replication should happen, make sure the new node is listed here.
		[replicationservers]
		mongo4
		mongo1
		mongo2

		#The list of mongodb configuration servers, make sure it is 1 or 3
		[mongocservers]
		mongo1
		mongo2
		mongo3

		#The list of servers where mongos servers would run. 
		[mongosservers]
		mongos

Make sure you have the new node added in the replicationservers section and execute the following command:

		ansible-playbook -i hosts playbooks/addnode.yml -e servername=mongo4

###Verification.

The verification of the newly added node can be as easy checking the sharding status and see the chunks being rebalanced to the newly added node.

			$/usr/bin/mongo localhost:8888/admin -u admin -p 123456
			mongos> sh.status()
				--- Sharding Status --- 
  				sharding version: { "_id" : 1, "version" : 3 }
  			shards:
			{  "_id" : "bensible",  "host" : "bensible/bensible:20103,web2:20103,web3:20103" }
			{  "_id" : "web2",  "host" : "web2/bensible:20105,web2:20105,web3:20105" }
			{  "_id" : "web3",  "host" : "web3/bensible:20102,web2:20102,web3:20102" }
			{  "_id" : "web4",  "host" : "web4/bensible:20101,web3:20101,web4:20101" }
  			databases:
			{  "_id" : "admin",  "partitioned" : false,  "primary" : "config" }
			{  "_id" : "test",  "partitioned" : true,  "primary" : "bensible" }
		
			test.test_collection chunks:
			
				web4	3
				web3	6
				web2	6
				bensible	5

    