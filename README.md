# PostgreSQL CDC Setup with Debezium

This guide walks you through setting up PostgreSQL Change Data Capture (CDC) using Debezium for logical replication. This setup will allow Debezium to capture changes from a source PostgreSQL database and replicate them to target databases or edge nodes.

## Steps

1. **Initialize PostgreSQL Database**  
   Run the following command to initialize the PostgreSQL database:  
   `initdb /opt/homebrew/var/postgresql@14`

2. **Create the Database for Capturing Changes**  
   Create a new database where Debezium will capture changes:  
   `CREATE DATABASE debezium_test_db;`

3. **Create a User for Debezium with Replication Privileges**  
   Create a user `postgres` with a password and grant replication privileges:  
   `CREATE USER postgres WITH PASSWORD 'debezium_test_password';`

4. **Grant Replication and Superuser Privileges**  
   Grant the necessary replication privileges to the user:  
   `ALTER USER postgres WITH REPLICATION;`  
   `ALTER USER postgres WITH SUPERUSER;`

5. **Grant Necessary Permissions on the Database**  
   Grant permissions on the `sourcedb` database for the `postgres` user:  
   `GRANT CONNECT ON DATABASE sourcedb TO postgres;`  
   `GRANT CREATE ON DATABASE sourcedb TO postgres;`  
   `GRANT USAGE ON SCHEMA public TO postgres;`

6. **Grant Permissions on Existing Tables**  
   Grant `SELECT`, `INSERT`, `UPDATE`, and `DELETE` permissions on all existing tables in the `public` schema:  
   `GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO postgres;`

7. **Grant Permissions on Future Tables**  
   Grant the same permissions on any future tables created in the `public` schema:  
   `ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO postgres;`

8. **Create a Publication for Logical Replication**  
   Create a publication for logical replication, allowing Debezium to capture changes:  
   `CREATE PUBLICATION dbz_publication FOR ALL TABLES;`

9. **Create a Logical Replication Slot**  
   Create a logical replication slot for Debezium to stream changes:  
   `SELECT pg_create_logical_replication_slot('debezium_slot', 'pgoutput');`

10. **Show the Location of the Configuration File**  
    Display the location of the PostgreSQL configuration file (`postgresql.conf`):  
    `SHOW config_file;`

11. **Show the Location of the HBA Configuration File**  
    Display the location of the `pg_hba.conf` file:  
    `SHOW hba_file;`

12. **Configure Replication Settings in `postgresql.conf`**  
    Ensure the following settings are configured in `postgresql.conf` (check the output from the previous step):  
    `wal_level = logical             # Required for logical replication`  
    `max_replication_slots = 4       # Adjust based on your needs`  
    `max_wal_senders = 4            # Adjust based on your needs`  
    `listen_addresses = '*'         # Listen on all network interfaces (or specify IP)`  
    `shared_preload_libraries = 'pgoutput'  # Only if you are using `pglogical` extension`

13. **Restart PostgreSQL**  
    Restart PostgreSQL for the changes to take effect.

14. **Verify the Status of the Replication Slot**  
    Verify the status of the logical replication slot:  
    `SELECT * FROM pg_replication_slots;`

15. **Allow Debezium to Connect for Logical Replication**  
    Modify the `pg_hba.conf` file to allow Debezium to connect for logical replication (replace `<Debezium_Server_IP>` with the actual IP address of your Debezium server):  
    `host    replication    debezium_test_user    <Debezium_Server_IP>/32    md5`


-----

# Monitoring Events in Kafka

This guide walks through how to monitor Kafka events and topic creations.

## Steps

### 1. Spin Up the Kafka Tools YAML File for Debug
Run the following command to apply the Kafka tools YAML file for debugging:
kubectl apply -f kafka-tools.yaml -n kafka

### 2. Check If Topics Are Created
To list all Kafka topics, run:
kafka-topics --list --bootstrap-server my-cluster-kafka-bootstrap:9093

This should show all our topics mentioned in the Kafka Connect Yaml file
- `offset.storage.topic: connect-offsets`   (Kafka topic to store connector offsets)
- `config.storage.topic: connect-configs`   (Kafka topic to store connector configuration)
- `status.storage.topic: connect-status`   (Kafka topic to store connector task statuses)

### 3. Create a Table in PostgreSQL
Execute the following SQL to create the `employees` table in PostgreSQL:
- `CREATE TABLE employees (
employee_id SERIAL PRIMARY KEY,
first_name VARCHAR(50) NOT NULL
);`

### 4. Check if a Topic for the Table Is Created with the Specified Prefix in Kafka Connector
The Kafka connector is configured with a `topic.prefix` (e.g., `"postgres-cluster"`). After creating the table and inserting values, the corresponding topic should be created in Kafka. 
- ie - `{prefix}-{schema_name}-{table_name}`

To list all Kafka topics, run:
`kafka-topics --list --bootstrap-server my-cluster-kafka-bootstrap:9093`

Example output:
- __consumer_offsets
- connect-configs
- connect-offsets
- connect-status
- postgres-cluster.public.employees

### 5. Consume the Messages
To consume the messages from the created Kafka topic, use the following command:
* `kafka-console-consumer --bootstrap-server my-cluster-kafka-bootstrap:9093 --topic postgres-cluster.public.employees --from-beginning`


