# Configuração do Liferay Cluster Link (Ambiente Produção – Sem Docker)

Este documento descreve como configurar o **Liferay Cluster Link** em um **ambiente de produção**, utilizando **múltiplas máquinas (VMs)** com banco de dados e Elasticsearch compartilhados.

O objetivo é permitir a comunicação segura entre os nós do Liferay utilizando **JGroups com TCP Unicast e autenticação X.509 (RSA)**.

> **Estratégia recomendada**  
> Iniciar os nós do Liferay com o Cluster Link desabilitado, gerar o certificado no primeiro nó, distribuir o keystore e arquivo de configuração TCP.xml para os demais nós e somente então habilitar o Cluster Link em todos.

---

## Visão Geral da Arquitetura

* Múltiplos nós do Liferay (VMs ou servidores físicos)
* Banco de dados compartilhado (ex.: DB2, PostgreSQL, Oracle)
* Elasticsearch compartilhado
* Comunicação de cluster via **JGroups TCP (unicast)**
* Autenticação de cluster usando **certificado X.509 (RSA)**

Exemplo:

```
liferay-node1   10.0.0.10
liferay-node2   10.0.0.11
liferay-node3   10.0.0.12
```

---

## Pré-requisitos de Infraestrutura

### Rede
* Comunicação TCP liberada entre todos os nós
* Portas do JGroups liberadas (ex.: 7800)
* DNS funcional ou IPs fixos

### Sistema
* Java compatível com a versão do Liferay
* `keytool` disponível
* Sincronização de horário (NTP)

---

## Passo 1 — Iniciar os Nós com Cluster Link Desabilitado

Em **todos os nós do Liferay**, edite:

```
[LIFERAY_HOME]/portal-ext.properties
```

Mantenha o Cluster Link desabilitado:

```properties
# cluster.link.enabled=true
# cluster.link.channel.properties.control=/opt/liferay/unicast/tcp.xml
# cluster.link.channel.properties.transport.0=/opt/liferay/unicast/tcp.xml
# cluster.link.auth.cert.alias=liferay-cluster-cert
# cluster.link.auth.keystore.path=/opt/liferay/unicast/cluster_link_keystore.jks
```

Inicie apenas o primeiro nó:

```bash
./tomcat/bin/startup.sh
```

---

## Passo 2 — Criar Estrutura de Arquivos

Em **todos os nós**:

```bash
sudo mkdir -p /opt/liferay/unicast
sudo chown -R liferay:liferay /opt/liferay/unicast
```

---

## Passo 3 — Gerar o Certificado no Primeiro Nó

```bash
keytool -genkeypair \
  -alias liferay-cluster-cert \
  -keyalg RSA \
  -keystore /opt/liferay/unicast/cluster_link_keystore.jks \
  -storepass TESTE$%3030
```

---

## Passo 4 — Distribuir o Keystore

```bash
scp /opt/liferay/unicast/cluster_link_keystore.jks liferay@liferay-node2:/opt/liferay/unicast/
scp /opt/liferay/unicast/cluster_link_keystore.jks liferay@liferay-node3:/opt/liferay/unicast/
```

---

## Passo 5 — Configurar JGroups TCP em nós

Criar arquivo em nós:

```
/opt/liferay/unicast/tcp.xml
```

```xml
<config
	xmlns="urn:org:jgroups"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="urn:org:jgroups http://www.jgroups.org/schema/jgroups.xsd"
>
	<TCP
		bind_port="7800"
		max_bundle_size="64K"
		recv_buf_size="${tcp.recv_buf_size:130k}"
		send_buf_size="${tcp.send_buf_size:130k}"
		sock_conn_timeout="300"
		thread_pool.keep_alive_time="30000"
		thread_pool.max_threads="20"
		thread_pool.min_threads="0"
	/>

  // Alterar aqui para utilizar os IPs reais      
  <TCPPING 		
    async_discovery="true"
    initial_hosts="${jgroups.tcpping.initial_hosts:10.0.0.10[7800],10.0.0.11[7800],10.0.0.12[7800]}"
    port_range="1"
  />

  <MERGE3
    max_interval="30000"
    min_interval="10000"
  />
	<FD_SOCK />
	<FD_ALL interval="3000" timeout="9000" />
	<VERIFY_SUSPECT timeout="1500" />
	<BARRIER />
	<pbcast.NAKACK2 use_mcast_xmit="false"
				   discard_delivered_msgs="true" />
	<UNICAST3 />
	<pbcast.STABLE desired_avg_gossip="50000"
				   max_bytes="4M" />
	<pbcast.GMS join_timeout="3000" print_local_addr="true" />
	<UFC
		max_credits="2M"
		min_threshold="0.4"
	/>
	<MFC
		max_credits="2M"
		min_threshold="0.4"
	/>
	<FRAG2 frag_size="60K" />
	<!--RSVP resend_interval="2000" timeout="10000"/-->
	<pbcast.STATE_TRANSFER />
</config>
```

---

## Passo 6 — Habilitar o Cluster Link nos nós

```properties
cluster.link.enabled=true
cluster.link.channel.properties.control=/opt/liferay/unicast/tcp.xml
cluster.link.channel.properties.transport.0=/opt/liferay/unicast/tcp.xml

cluster.link.auth.cert.alias=liferay-cluster-cert
cluster.link.auth.cert.password=TESTE$%3030
cluster.link.auth.cipher.type=RSA
cluster.link.auth.keystore.password=TESTE$%3030
cluster.link.auth.keystore.path=/opt/liferay/unicast/cluster_link_keystore.jks
cluster.link.auth.keystore.type=JKS
cluster.link.auth.value=TESTE$%3030

web.server.display.node=true
```

---

## Passo 7 — Reiniciar os Nós

```bash
./tomcat/bin/startup.sh
```

---

## Passo 8 — Validação

Nos logs:

```
Accepted view
```