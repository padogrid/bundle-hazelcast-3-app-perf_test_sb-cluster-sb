# Split-Brain Cluster

This bundle provides scripts, configuration files, and apps for creating a network split-brain environment where you can test Hazelcast's split-brain capabilities. Included are

- Pod **pod-sb**. The pod-sb pod is configured with one (1) primary node and five (5) nodes. Each node has 2 GiB of memory.

- Cluster **sb**. The sb cluster is configured with five (5) VM members running in the `pod-sb` pod. It includes scripts that use `iptables` to drop TCP packets to split the `sb` cluster into two (2). It is configured with split-brain quorum rules for th following maps.

  - nw/customers
  - nw/orders

- App **perf_test_sb**. The `perf_test_sb` app is configured to run on a split cluster.

- App **desktop**. The `desktop` app is used to compare data in the split clusters and the merged cluster. It is not included in the bundle because the vanilla desktop works without any modifications. You will be installing the `desktop` app as one of the steps shown in the [Creating Split-Brain](#creating-split-brain) section.

*Note that the `sb` cluster is configured to run in the `pod-sb` pod with its members running as VM hosts and not Vagrant pod hosts.*

## Installation Steps

We will be building the following components as we setup and run the environment.

1. Configure pod
2. Build pod
3. Build `perf_test_sb`
4. Build `desktop`

Follow the instructions in the subsequent sections.

### Configuring Pod

You must first build the `pod-sb` pod, which has been created with the IP addresses shown below. Make sure you have the host-only private network created for 192.168.56.1. For instructions see 

[Creating VirtualBox Private Network (host-only)](https://github.com/javapark1/hazelcast-addon/tree/master/hazelcast-addon-deployment/src/main/resources/pods#creating-virtualbox-private-network-host-only)

| Name                 | IP/Mask         |
| -------------------- | --------------- |
| Host Network Adapter | 192.168.56.1/24 |
| pnode.local          | 192.168.56.10   |
| node-01.local        | 192.168.56.11   |
| node-02.local        | 192.168.56.12   |
| node-03.local        | 192.168.56.13   |
| node-04.local        | 192.168.56.14   |
| node-05.local        | 192.168.56.15   |

The pod-sb has been configured in the `etc/pod.properties` as shown below.

**etc/pod.properties:**

```properties
# Pod name
pod.name=pod-sb
pod.type=vagrant
pod.box.image=ubuntu/trusty64

# Node (guest) properties.
node.name.primary=pnode
node.name.prefix=node
node.ip.lastOctet=10
node.memory.primary=2048
node.memory.data=2048
node.count=5

# Host properties.
host.productsDir=/Users/dpark/Work/Hazelcast/workspaces_0.2.4-SNAPSHOT/ws-pods/products
```

Make sure to change the `host.productsDir` value to reflect your Linux product directory path. You can also change the memory size of the primary and data nodes if you don't have enough memory. If you change the memory size then you may also need to change the member heap size, which is also mentioned in the [Configuring Cluster](#configuring-cluster) section below. 

The products directory must contain Linux version of JDK and Hazelcast Enterprise. For example,

```console
products
├── hazelcast-enterprise-3.12.5
└── jdk1.8.0_212
```

To change the pod properties:

```console
cd_pod pod-sb
vi etc/pod.properties
```

Once you are satisfied with the `pod-sb` properties, run `build_pod` to build the nodes.
 
```console
# Build and start the pod
build_pod -pod pod-sb
```

### Configuring Cluster

If you changed the memory size of the primary and data nodes in the [Configuring Pod](#configuring-pod) section, then you can adjust the Hazelcast member min/max heap size in the `etc/cluster.properties` file as follows:

```console
switch_cluster sb
vi etc/cluster.properties

# Heap min and max values in etc/cluster.properties
heap.min=1g
heap.max=1g
```

## Creating Split-Brain

### 1. Start cluster and Management Center

Login to `pnode.local` and start the `sb` cluster as follows:

```console
cd_pod pod-sb
vagrant ssh
password: vagrant

# Once logged in to Varant VM, pnode, execute the following:
switch_cluster sb
start_cluster
start_mc
```

### 2. Monitor the Management Center from your web browser

[http://pnode.local:8080/hazelcast-mancenter](http://pnode.local:8080/hazelcast-mancenter)

### 3. Ingest data - perf_test_sb

From your host OS, build `perf_test_sb` and run `test_group` as follows:

```console
cd_app perf_test_sb; cd bin_sh
./build_app
./test_group -prop ../etc/group-factory.properties -run
```

### 4. View data - desktop

From your host OS, install and run the `desktop` app as follows:

```console
create_app -app desktop -name desktop_sb
cd_app desktop_sb; cd bin_sh
./build_app
cd ../hazelcast-desktop_<version>/bin_sh
./desktop
```

You can enter any of the member addresses for the `Locators` text field when you login from the desktop. For example, `node-01.local:5701` connects to the `node-01.local` member. `User Name` is required and can be any name. `Password` is optional and ignored.

```console
Locators: node-01.local:5701
App ID: sys
User Name: foo
Password: <leave blank>
```

### 5. Split cluster

From `pnode.local`, run `split_cluster` as follows:

```console
switch_cluster sb; cd bin_sh
./split_cluster
```

To see the `iptables` rules set by `split_cluster`, run `list_rules` as follows:

```console
./list_rules
```

### 6. From `pnode.local`, monitor the log files to see the cluster splits into two (2) as follows:

| Cluster | Nodes                                       |
| ------- | ------------------------------------------- |
| A       | node-01.local, node-02.local                |
| B       | node-03.local, node-04.local, node-05.local |

```console
# See Cluster A (view node-01.local or node-02.local log file)
show_log

# See Cluster B (view node-03.local, node-04.local, node-05.local log file)
show_log -num 5
```

You should see something like the following in the log files:

**Cluster A**

```console
Members {size:2, ver:8} [
	Member [192.168.56.11]:5701 - 49a65895-221d-4b26-b562-7c53eec092e1 this
	Member [192.168.56.12]:5701 - 17fcfc4a-5593-47be-bde1-a4f5ed3e50b2
]
```

**Cluster B**

```console
Members {size:3, ver:6} [
	Member [192.168.56.15]:5701 - 9c41e05d-1acf-449b-a25f-1991c8756617 this
	Member [192.168.56.14]:5701 - 235691e0-bc36-407e-971a-f36189fc0801
	Member [192.168.56.13]:5701 - 3a3c8639-0b7b-4f4f-9daf-e262160126b9
]
```

Try refreshing the management center from your browser. You should see the list of members changing sporadically indicating there is a network issue.

### 7. Ingest data into Cluster A - `perf_test_sb`

From your host OS, run `test_group` which has been preconfigured to connect to Cluster B, i.e., node-03.local, node-04.local, node-05.local (see `etc/hazelcast-client.xml`). `test_group` updates the data that was inserted earlier. We'll compare the new data with the old data in the split clusters.

```console
cd_app perf_test_sb; cd bin_sh
./test_group -prop ../etc/group-factory.properties -run
```

### 8. Compare data between Cluster A and Cluster B

   - Launch desktop and login to Cluster A, e.g., `node-01.local:5701`
   - Launch desktop and login to Cluster B, e.g., `node-05.local:5701`

```console
cd_app desktop_sb
cd hazelcast-desktop_<version>/bin_sh

# Launch two (2) instances of desktop
./desktop
./desktop
```

### 9. Execute queries on both desktop instances

Execute queries on both desktop instances so that we can compare the results later when we merge the clusters.

```console
--- From each desktop instacne, execute the following queries
--- (Note that the desktop supports SQL comments):
select * from nw/customers order by customerId;
select * from nw/orders order by orderId;
```

When you execute the above queries you should see the following behavior:

Map          | Cluster A                 | Cluster B     |
------------ | ------------------------- | ------------- |
nw/customers | Query Success             | Query Success |
nw/orders    | Query Failure (Exception) | Query Success |

**Query Exception:** *com.hazelcast.quorum.QuorumException: Split brain protection exception: quorumRuleWithThreeMembers has failed!*

The exception occurs in Cluster A because the split-brain quorum size for the `nw/customers` map is configured with two (2) as follows (See `etc/hazelcast.xml`):

```xml
   <quorum name="quorumRuleWithTwoMembers" enabled="true">
      <quorum-size>2</quorum-size>
   </quorum>
   <quorum name="quorumRuleWithThreeMembers" enabled="true">
      <quorum-size>3</quorum-size>
   </quorum>

   <map name="nw/customers">
      <merge-policy batch-size="100">LatestUpdateMergePolicy</merge-policy>
      <quorum-ref>quorumRuleWithTwoMembers</quorum-ref>
   </map>
   <map name="nw/orders">
      <merge-policy batch-size="100">LatestUpdateMergePolicy</merge-policy>
      <quorum-ref>quorumRuleWithThreeMembers</quorum-ref>
   </map>
```

You can view the `hazelcast.xml` file as follows:

```console
cd_cluster sb
less etc/hazelcast.xml
```

### 10. Merge clusters

From `pnode.local`, run `merge_cluster` as follows:

```console
switch_cluster sb; cd bin_sh
./merge_cluster
```

### 11. Monitor merged cluster

```console
show_log
```

Upon a successful merge, which takes about a minute, you should see something like the following in all the log files:

```console
Members {size:5, ver:11} [
	Member [192.168.56.15]:5701 - 9c41e05d-1acf-449b-a25f-1991c8756617
	Member [192.168.56.14]:5701 - 235691e0-bc36-407e-971a-f36189fc0801
	Member [192.168.56.13]:5701 - 3a3c8639-0b7b-4f4f-9daf-e262160126b9
	Member [192.168.56.12]:5701 - c10d17a9-fe0c-4c22-af7a-67f8dd702b9c
	Member [192.168.56.11]:5701 - 79d0fa6e-d7dd-46d4-8525-52250dee6e80 this
]
```

The management center should also show five (5) members.

### 12. Compare data between the merged cluster and Cluster B

The merged cluster should have the exact same data as Cluster B since both maps are configured with `LatestUpdateMergePolicy`.

## Tearing Down

### Stop Cluster

From `pnode.local`, execute the following:

```console
stop_cluster -cluster sb
stop_mc -cluster sb
```

### Stop Pod

From your host OS, execute the following:

```console
stop_pod -pod pod-sb
```

### Remove Pod

From you host OS, execute the following:

```console
remove_pod -pod pod-sb
```

### Close Desktop

Close `desktop` app instances by clicking on the close icon.
