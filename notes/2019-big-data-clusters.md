# Big data cluster notes

E1009 14:31:46.753798   86216 start.go:152] Error starting host:  Error starting stopped host: Unable to start the VM: /usr/local/bin/VBoxManage startvm minikube --type headless failed:
VBoxManage: error: The virtual machine 'minikube' has terminated unexpectedly during startup with exit code 1 (0x1)
VBoxManage: error: Details: code NS_ERROR_FAILURE (0x80004005), component MachineWrap, interface IMachine

Configured with too many resources, lower the amount of memory
Can you up the maximum allowed by Hyper-V?

https://github.com/Microsoft/sql-server-samples/tree/master/samples/features/sql-big-data-cluster
https://docs.microsoft.com/en-us/sql/big-data-cluster/tutorial-query-hdfs-storage-pool?view=sqlallproducts-allversions
https://docs.microsoft.com/en-us/sql/big-data-cluster/train-and-create-machinelearning-models-with-spark?view=sqlallproducts-allversions
https://docs.microsoft.com/en-us/sql/big-data-cluster/export-model-with-spark-mleap?view=sqlallproducts-allversions
https://docs.microsoft.com/en-us/sql/big-data-cluster/train-and-create-machinelearning-models-with-spark?view=sqlallproducts-allversions
https://docs.microsoft.com/en-us/sql/big-data-cluster/export-model-with-spark-mleap?view=sqlallproducts-allversions

http://www.nielsberglund.com/2018/11/10/sql-server-2019-big-data-cluster-on-azure-kubernetes-service/

Minimum of 16-24GB of memory

Niels Berglund
I started with 8 Gb of memory assigned to minikube, I changed that to 16Gb, per recommendation of Umachandar Jayachandran (UC).
I started with Python 3.7, but downgraded to 3.5. This due to errors with Requests, and UC said he had had success using 3.5. Now I do no think that matters as at the latest installation (success) I still had some errors related to Requests
Initially I had Docker for Windows installed with Kubernetes enabled. It was not until I had un-installed Docker (and therefore Kubernetes) that my install succeeded. When I feel brave I will install Docker for Windows again (without Kubernetes) and ensure it works.
Finally the successful installation took 2.5 hours: from mssqlctl create cluster to Controller pod is running - 14 min
Controller pod is running to Creating cluster with name: 1 minute
Creating cluster with name to Master pool is ready: 30 minutes
Master pool is ready to Cluster deployed successfully: 1 hour 45 minutes
Oh, I forgot I also changed the environment variables for replicas as per Travis Wright to:
SET CLUSTER_COMPUTE_POOL_REPLICAS=1
SET CLUSTER_STORAGE_POOL_REPLICAS=1
SET CLUSTER_DATA_POOL_REPLICAS=1

Travis Wright:
My first guess is the formatting of your SET statements which includes a space after the equals sign in some cases.  Try this instead: 

Travis Wright:
We will keep working on simplifying the installation experience.  Keep in mind though that you are not installing a single feature on a single server.  You are deploying an entire cluster of an Always On Availability Group SQL Server instance for the master (coming soon), a cluster of n number of SQL Server instances for compute pool, a cluster of n number of SQL Server instances for data pool, a storage pool with n number of Spark, SQL Server instances + HDFS, HDFS name node with HA, Spark driver, Elastic Search, InfluxDB, agents on every pod and every Kubernetes node, an admin portal, a controller service with API, a controller SQL Server instance, YARN UI, Spark UI, Kibana, Granfana, etc, etc. and it's all integrated and wired up automatically when everything is done running.  If you step back and think of it that way, it's actually pretty amazing that it can be deployed with a single command. :)

kubectl get pods -n


https://docs.microsoft.com/en-us/sql/big-data-cluster/concept-master-instance?view=sql-server-ver15
SQL Server machine learning services is an add-on feature to the database engine, used for executing Java, R and Python code in SQL Server. This feature is based on the SQL Server extensibility framework, which isolates external processes from core engine processes, but fully integrates with the relational data as stored procedures, as T-SQL script containing R or Python statements, or as Java, R or Python code containing T-SQL.

## Outstanding issues

- When using pipe delimited external file_format
Msg 46506, Level 16, State 50, Line 4
Invalid set of options specified for 'FILE_FORMAT'.