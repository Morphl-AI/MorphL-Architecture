## The Mechanics Of The Platform

### Every morning:  
The GA connector will run, extracting all data from the previous day and loading it into the raw_activity_stream_data table in Cassandra.

This task will be controlled by AirFlow.

### Every week:  
The preprocessor will run, extracting features from the activity stream of each user.  
Important to note is the fact that activity streams are deemed to fall into one of four categories:  
Type 1) activity streams of users who were active at some point long ago but have churned;  
Type 2) activity streams of veteran users who are currently still showing high levels of engagement;  
Type 3) activity streams of recent users about whom enough data was collected to make a prediction;  
Type 4) activity streams of very recent users about whom not enough data is yet available.  
The preprocessor will ignore type 4 activity streams for now, but will begin handling them as soon as they turn into type 3 ones.

The preprocessor will extract features for types 1, 2 and 3.  
As well, it will assign the label "churned" to type 1 and the label "engaged" to type 2.  
The preprocessor will load features and labels for types 1 and 2 into the veteran_users table in Cassandra.  
It will also load features (but no labels) for type 3 activity streams into the current version of the recent_users table in Cassandra.

This task will be controlled by an AirFlow DAG.

### Every week:  
Upon successful completion of the preprocessing task, the veteran_users table in Cassandra now contains features and labels that TensorFlow can use to train a model.  
At this point, AirFlow will launch an instance of a Docker container with a TensorFlow model training routine, which will ingest the features and labels in veteran_users and will save a model file for later use.

### Every week:  
Upon successful completion of the TensorFlow model training routine, AirFlow will launch an instance of a Docker container with a TensorFlow ModelServer, which will ingest the features in the current version of the recent_users and make use of the model file that was saved at the end of the training run.  
At the end of this batch prediction run, the resulting predictions will be loaded into the "prediction" column of the current version of the recent_users table in Cassandra.

### Every week:  
Upon successful completion of the TensorFlow ModelServer batch prediction run, AirFlow will initiate the Kubernetes commands necessary to deploy a new REST API endpoint, pointing it to the current version of the recent_users table in Cassandra, and supplying the endpoint with a freshly generated API key.  
It is important to mention that the REST API endpoints pertaining to previous weeks are still up and running at this point in time.  
As well, when we depict "the recent_users table in Cassandra" in the architectural diagram, what we mean is not a single table, but a family of versions of the recent_users table, all with the same schema, but with a year-and-week-number suffix attached to the name, indicating when the table was generated and populated (for example: recent_users_2018_27).

As the reader might infer from the above, the ETL pipeline is orchestrated by Airflow and relies on Docker containers as well as on a Kubernetes cluster.
