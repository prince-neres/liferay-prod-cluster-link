# Liferay Cluster Link Setup (Docker Environment)

This document describes how to configure **Liferay Cluster Link** using **Docker containers**, based on a multi-node Liferay 7.4 environment with a shared database and Elasticsearch.

The key idea is:

> **Start the containers with Cluster Link disabled, generate the certificate on the first Liferay node, copy it to the other nodes, and only then enable Cluster Link.**

---

## Environment Overview

* Docker Compose based setup
* Multiple Liferay nodes
* Shared database (DB2 in this case)
* Shared Elasticsearch
* Cluster communication using **JGroups + X509 (RSA)**

---

## Step 1 — Start Containers with Cluster Link Disabled

Initially, **Cluster Link must NOT be enabled**.

In the `portal-ext.properties` file of **all Liferay nodes**, keep the Cluster Link properties commented out or not present:

```properties
# ## CLUSTER LINK ##
# cluster.link.enabled=true
# cluster.link.autodetect.address=postgres:5432
# cluster.link.channel.properties.control=jgroups/secure/x509/udp_control.xml
# cluster.link.channel.properties.transport.0=jgroups/secure/x509/udp_transport.xml
# cluster.link.auth.cert.alias=liferay-node1-cert
# cluster.link.auth.cert.password=TESTE$%3030
# cluster.link.auth.cipher.type=RSA
# cluster.link.auth.keystore.password=TESTE$%3030
# cluster.link.auth.keystore.path=/opt/liferay/cluster_link_keystore.jks
# cluster.link.auth.keystore.type=JKS
# cluster.link.auth.value=TESTE$%3030
# web.server.display.node=true
```

> **Why?**
> At this point, the keystore does not exist yet. If Cluster Link is enabled without the certificate, Liferay will fail to start.

Start the containers normally:

```bash
docker compose up -d
```

---

## Step 2 — Generate the Certificate on the First Liferay Node

Access the **first Liferay container**:

```bash
docker exec -it liferay-node1 bash
```

Generate the keystore **inside the container**:

```bash
keytool -genkeypair \
  -alias liferay-node1-cert \
  -keyalg RSA \
  -keystore /opt/liferay/cluster_link_keystore.jks \
  -storepass TESTE$%3030
```

### Important Notes

* The keystore path **must match** the property:

  ```properties
  cluster.link.auth.keystore.path=/opt/liferay/cluster_link_keystore.jks
  ```
* The alias **must match**:

  ```properties
  cluster.link.auth.cert.alias=liferay-node1-cert
  ```

Verify the file:

```bash
ls -lh /opt/liferay/cluster_link_keystore.jks
```

---

## Step 3 — Copy the Keystore to the Other Liferay Nodes

All Liferay nodes must use **the same keystore**.

### Option 1 — Manual Copy Between Containers

```bash
docker cp liferay-node1:/opt/liferay/cluster_link_keystore.jks ./cluster_link_keystore.jks

docker cp ./cluster_link_keystore.jks liferay-node2:/opt/liferay/cluster_link_keystore.jks
```

---

### Option 2 — Shared Docker Volume (Recommended)

Mount a shared volume in the `docker-compose.yml`:

```yaml
volumes:
  - ./bundles/cluster:/opt/liferay/cluster
```

Then update the property:

```properties
cluster.link.auth.keystore.path=/opt/liferay/cluster/cluster_link_keystore.jks
```

This guarantees that all nodes always use the same keystore.

---

## Step 4 — Enable Cluster Link

After the keystore is available on **all nodes**, enable Cluster Link by adding (or uncommenting) the following properties in `portal-ext.properties` of **every Liferay node**:

```properties
## CLUSTER LINK ##
cluster.link.enabled=true
cluster.link.autodetect.address=postgres:5432

cluster.link.channel.properties.control=jgroups/secure/x509/udp_control.xml
cluster.link.channel.properties.transport.0=jgroups/secure/x509/udp_transport.xml

cluster.link.auth.cert.alias=liferay-node1-cert
cluster.link.auth.cert.password=TESTE$%3030
cluster.link.auth.cipher.type=RSA

cluster.link.auth.keystore.password=TESTE$%3030
cluster.link.auth.keystore.path=/opt/liferay/cluster_link_keystore.jks
cluster.link.auth.keystore.type=JKS
cluster.link.auth.value=TESTE$%3030

web.server.display.node=true
```

### Notes

* `cluster.link.autodetect.address` can be any shared and reachable address
* All passwords, aliases and keystore paths **must be identical across nodes**
* `web.server.display.node=true` helps identify which node is serving requests

---

## Step 5 — Restart Liferay Nodes

Restart the Liferay containers:

```bash
docker compose restart liferay-node1 liferay-node2
```

---

## Step 6 — Validation

Check the Liferay logs. You should see messages similar to:

```text
Accepted view
```

✅ Cluster Link is now correctly configured for a Docker-based Liferay environment.
