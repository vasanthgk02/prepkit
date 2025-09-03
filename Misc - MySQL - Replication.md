# MySQL Replication & MariaDB Galera Cluster — Complete Notes

## 1. What is MySQL Replication?
MySQL replication allows you to copy data from one database server (Primary) to others (Replicas) for:

- Load balancing (read/write separation)
- High availability
- Backup and disaster recovery

## 2. Types of MySQL Replication

| Replication Type    | Description |
|---------------------|-------------|
| Asynchronous        | Default. Master doesn’t wait for replicas to confirm receipt or apply logs. |
| Semi-synchronous    | Master waits for at least one replica to receive the transaction log. |
| Group Replication   | Built-in quorum-based multi-node cluster. Stronger consistency. |

## 3. Replication Threads (Internals)

**On the Primary:**  
- **Binlog Writer**: Writes all changes to the binary log (mysql-bin.000001, etc.)

**On the Replica:**  
- **IO Thread**:  
  - Connects to master  
  - Reads binlog events  
  - Writes them to the relay log

- **SQL Thread**:  
  - Reads from relay log  
  - Executes SQL changes to apply them to the replica

🧠 These threads run continuously to keep the replica up to date.

## 4. Setting Up Basic MySQL Asynchronous Replication

### 🛠 On Primary (`my.cnf`):
```ini
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
```

```sql
CREATE USER 'replica_user'@'%' IDENTIFIED BY 'replica_password';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';
FLUSH PRIVILEGES;
```

### 🛠 Get current binlog info:
```sql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS; -- Note File & Position
```

### 🛠 Dump and import data:
```bash
mysqldump --all-databases --master-data > dump.sql
mysql < dump.sql  # on replica
```

### 🛠 On Replica (`my.cnf`):
```ini
server-id = 2
relay_log = mysql-relay-bin
read_only = 1
```

### 🛠 Configure replication:
```sql
CHANGE MASTER TO
  MASTER_HOST='primary_ip',
  MASTER_USER='replica_user',
  MASTER_PASSWORD='replica_password',
  MASTER_LOG_FILE='mysql-bin.000003',
  MASTER_LOG_POS=12345;

START SLAVE;
```

### 🛠 Verify:
```sql
SHOW SLAVE STATUS\G
-- Look for:
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: 0
```

## 5. Replication Lag and Inconsistency

Why replicas can be out of sync:

- Network latency
- Long-running queries
- Delayed SQL thread
- Crash or server overload

🛑 Replication is eventually consistent, not immediately.

## 6. Making Replication More Consistent

✅ Use Semi-Synchronous Replication:

### 🔧 On Primary:
```sql
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
```

### 🔧 On Replica:
```sql
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

❗ But this still does not wait for replica to apply the transaction.

## 7. Limitations of MySQL Replication

| Weakness                 | Explanation |
|--------------------------|-------------|
| No strong consistency    | Replica may lag or crash before applying logs |
| Hard failover            | Manual promotion is complex |
| No native synchronous replication | Only semi-sync with limitations |

## 8. Introducing Galera Cluster (MariaDB, Percona)

✅ What is Galera?  
Galera is a plugin that provides:

- True synchronous replication
- Multi-master support
- No replication lag
- Automatic conflict handling

### 🔁 How Galera Works:
- A client writes to any node.
- The write is broadcast to all nodes.
- All nodes certify and apply the transaction.
- Only then does the client get a success response.

### 🧱 Galera Topology:
```
      [Node 1] ◀────▶ [Node 2]
         ▲                ▲
         │                │
         └────▶ [Node 3] ◀────┘
```
All nodes are equal. All see the same data.

## 9. MariaDB vs MySQL — Key Differences

| Feature               | MySQL                               | MariaDB (with Galera)                 |
|------------------------|--------------------------------------|----------------------------------------|
| Replication type       | Async / Semi-sync / Group            | Galera (true sync)                     |
| Multi-master support   | Limited (Group Replication)          | ✅ Full multi-master                   |
| Read consistency       | Eventual                             | Strong (linearizable)                  |
| Slave lag              | Possible                             | None                                   |
| Conflict resolution    | Manual                               | Automatic (write-set detection)        |
| Open source            | Owned by Oracle                      | 100% community-driven                  |

## 10. Summary Table: Replication Models

| Type             | Waits For                  | Consistency | Lag   | Multi-Master |
|------------------|-----------------------------|-------------|-------|---------------|
| Async            | Nothing (fire & forget)     | Weak        | High  | No            |
| Semi-sync        | Replica receives logs       | Medium      | Low   | No            |
| MySQL Group Rep  | Quorum on receipt           | Medium+     | Low   | Partial       |
| Galera Cluster   | Quorum + apply confirmation | Strong      | None  | ✅ Yes         |

## 11. When to Use What?

| Use Case                          | Choose This                    |
|-----------------------------------|--------------------------------|
| Simple replication, low complexity | MySQL async                    |
| Stronger consistency, no data loss | MySQL semi-sync               |
| Read-after-write consistency needed | MariaDB with Galera Cluster |
| Multi-master and HA requirements   | MariaDB Galera or Percona XtraDB |

## 12. Tools and Monitoring

- `SHOW SLAVE STATUS\G`: Monitor MySQL replication
- `SHOW STATUS LIKE 'rpl_semi%'`: Monitor semi-sync
- `SHOW GLOBAL STATUS LIKE 'wsrep_%'`: For Galera status
- `pt-heartbeat` (Percona Tool): Monitor replication lag

## 13. Final Notes

- MySQL is great for basic use cases, read replicas, and easy deployment.
- MariaDB + Galera is ideal for mission-critical apps needing consistency and uptime.
- Replication threads (IO + SQL) are key to how MySQL syncs — understanding these helps debug issues.
- Galera is an operational game-changer for high availability and consistency.