### **Step-by-Step Guide to Install and Configure a 3-Node Apache Cassandra Cluster on AWS Amazon Linux 2023**

---

## **Step 1: Provision 3 AWS EC2 Instances**
- **AMI**: Choose **Amazon Linux 2023 (AL2023)**
- **Instance Type**: `t3.medium` or `t3.large` (at least **4GB RAM**)
- **Storage**: 20GB (GP3 SSD recommended)
- **Networking**: Assign static **private IPs** or use AWS VPC with **private subnets**.

### **Security Group Configuration**
Allow inbound traffic on the following **ports**:
- **7000** (TCP) – Internal cluster communication
- **7001** (TCP) – Secure cluster communication
- **7199** (TCP) – JMX monitoring
- **9042** (TCP) – CQL client queries
- **9160** (TCP) – Thrift service (deprecated but may be needed)
- **22** (TCP) – SSH access

---

## **Step 2: Install Dependencies (Java & Cassandra)**
### **1. Install Java 11**
```sh
sudo dnf update -y
sudo dnf install -y java-11-amazon-corretto
java -version
```

### **2. Add Apache Cassandra Repository**
```sh
echo "[cassandra]
name=Apache Cassandra
baseurl=https://downloads.apache.org/cassandra/redhat/40x/
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://downloads.apache.org/cassandra/KEYS" | sudo tee /etc/yum.repos.d/cassandra.repo
```

### **3. Install Apache Cassandra**
```sh
sudo dnf install -y cassandra
```

### **4. Start and Enable Cassandra Service**
```sh
sudo systemctl start cassandra
sudo systemctl enable cassandra
```

Verify installation:
```sh
nodetool status
```
Expected Output (single node):
```
Datacenter: DC1
==================
Status=Up/Down
State=Normal/Leaving/Joining/Moving
Address      Load       Tokens  Owns    Host ID                               Rack
10.0.1.1     120MB      256     100%   abc123-def456                        rack1
```

---

## **Step 3: Configure the 3-Node Cassandra Cluster**
### **1. Edit Cassandra Configuration (`cassandra.yaml`)**
Modify Cassandra’s configuration on **all three nodes**:

```sh
sudo nano /etc/cassandra/cassandra.yaml
```

#### **Update Cluster Name**
```yaml
cluster_name: 'MyCassandraCluster'
```

#### **Set Seed Nodes (Choose Two Nodes)**
- Pick **two nodes** as **seed nodes** and configure them:
```yaml
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "10.0.1.1,10.0.1.2"
```
- Replace **10.0.1.1,10.0.1.2** with the **private IPs** of the first two nodes.

#### **Set Node’s Private IP Address**
On **each node**, configure its respective private IP:
```yaml
listen_address: 10.0.1.X  # Replace with each node's private IP
rpc_address: 0.0.0.0
broadcast_rpc_address: 10.0.1.X
```

#### **Configure Datacenter & Rack**
Edit the datacenter/rack settings on each node:
```sh
sudo nano /etc/cassandra/cassandra-rackdc.properties
```
Set the **same datacenter name** on all nodes:
```
dc=DC1
rack=RACK1
```

Save and exit.

---

## **Step 4: Restart Cassandra and Verify Cluster**
### **1. Restart Cassandra Service on All Nodes**
```sh
sudo systemctl restart cassandra
```

### **2. Verify Cluster Formation**
Run on **any node**:
```sh
nodetool status
```

Expected Output (3-node cluster):
```
Datacenter: DC1
==================
Status=Up/Down
State=Normal/Leaving/Joining/Moving
Address      Load       Tokens  Owns    Host ID                               Rack
10.0.1.1     120MB      256     33.3%   abc123-def456                        rack1
10.0.1.2     140MB      256     33.3%   ghi789-jkl012                        rack1
10.0.1.3     110MB      256     33.3%   mno345-pqr678                        rack1
```
- `UN` means **Up & Normal** (expected).
- Each node owns **~33%** of the data.

---

## **Step 5: Create Keyspace & Tables**
### **1. Connect to Cassandra**
```sh
cqlsh 10.0.1.1
```

### **2. Create a Keyspace**
```sql
CREATE KEYSPACE my_keyspace
WITH replication = {'class': 'NetworkTopologyStrategy', 'DC1': 3};
```

### **3. Create a Table**
```sql
CREATE TABLE my_keyspace.users (
    user_id UUID PRIMARY KEY,
    name TEXT,
    age INT
);
```

### **4. Insert Sample Data**
```sql
INSERT INTO my_keyspace.users (user_id, name, age) VALUES (uuid(), 'Alice', 30);
```

### **5. Query Data**
```sql
SELECT * FROM my_keyspace.users;
```

---

## **Step 6: Enable Auto-Start on Reboot**
```sh
sudo systemctl enable cassandra
```

---

## **Optional: Configure Performance Tuning**
### **1. Tune JVM Heap Size**
Edit JVM settings:
```sh
sudo nano /etc/cassandra/jvm-server.options
```
Modify these values (for `t3.large`):
```
-Xms4G
-Xmx8G
```

### **2. Enable Auto Compaction**
```sh
nodetool compact
```

---

