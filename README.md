# Configuração do Liferay Cluster Link (Ambiente Docker)

Este documento descreve como configurar o **Liferay Cluster Link** utilizando **containers Docker**, com base em um ambiente multi-nó do Liferay 7.4 com banco de dados e Elasticsearch compartilhados.

A ideia principal é:

> **Iniciar os containers com o Cluster Link desabilitado, gerar o certificado no primeiro nó do Liferay, copiá-lo para os outros nós e somente então habilitar o Cluster Link.**

---

## Visão Geral do Ambiente

* Setup baseado em Docker Compose
* Múltiplos nós do Liferay
* Banco de dados compartilhado (DB2 neste caso)
* Elasticsearch compartilhado
* Comunicação de cluster usando **JGroups + X509 (RSA)**

---

## Passo 1 — Iniciar os Containers com o Cluster Link Desabilitado

Inicialmente, o **Cluster Link NÃO deve estar habilitado**.

No arquivo `portal-ext.properties` de **todos os nós do Liferay**, mantenha as propriedades de Cluster Link comentadas ou ausentes:

```properties
# # Cluster Link
# cluster.link.enabled=true
# cluster.link.autodetect.address=
# # TCP unicast JGroups
# cluster.link.channel.properties.control=/opt/liferay/unicast/tcp.xml
# cluster.link.channel.properties.transport.0=/opt/liferay/unicast/tcp.xml
# # Segurança X.509
# cluster.link.auth.cert.alias=liferay-node1-cert
# cluster.link.auth.cert.password=TESTE$%3030
# cluster.link.auth.cipher.type=RSA
# cluster.link.auth.keystore.password=TESTE$%3030
# cluster.link.auth.keystore.path=/opt/liferay/unicast/cluster_link_keystore.jks
# cluster.link.auth.keystore.type=JKS
# cluster.link.auth.value=TESTE$%3030
# # Exibir o nó no rodapé
# web.server.display.node=true
```

> **Por quê?**  
> Neste ponto, o keystore ainda não existe. Se o Cluster Link for habilitado sem o certificado, o Liferay falhará ao iniciar.

Inicie os containers normalmente:

```bash
docker compose up -d
```

---

## Passo 2 — Gerar o Certificado no Primeiro Nó do Liferay

Acesse o **primeiro container do Liferay**:

```bash
docker exec -it liferay-node1 bash
```

Gere o keystore **dentro do container**:

```bash
keytool -genkeypair   -alias liferay-node1-cert   -keyalg RSA   -keystore /opt/liferay/unicast/cluster_link_keystore.jks   -storepass TESTE$%3030
```

### Observações Importantes

* O caminho do keystore **deve ser o mesmo** configurado na propriedade:

  ```properties
  cluster.link.auth.keystore.path=/opt/liferay/unicast/cluster_link_keystore.jks
  ```

* O alias **deve ser exatamente o mesmo**:

  ```properties
  cluster.link.auth.cert.alias=liferay-node1-cert
  ```

Verifique se o arquivo foi criado:

```bash
ls -lh /opt/liferay/unicast/cluster_link_keystore.jks
```

Esse diretório já está mapeado nos containers liferay, para que ambos utilizem o mesmo certificado e arquivo de configuração `tcp.xml` na pasta `/unicast`

## Passo 4 — Habilitar o Cluster Link

Após o keystore estar disponível em **todos os nós**, habilite o Cluster Link descomentando as seguintes propriedades no `portal-ext.properties` e, `/files`:

```properties
# Cluster Link
cluster.link.enabled=true
cluster.link.autodetect.address=

# TCP unicast JGroups
cluster.link.channel.properties.control=/opt/liferay/unicast/tcp.xml
cluster.link.channel.properties.transport.0=/opt/liferay/unicast/tcp.xml

# X.509 Security (se você quer segurança)
cluster.link.auth.cert.alias=liferay-node1-cert
cluster.link.auth.cert.password=TESTE$%3030
cluster.link.auth.cipher.type=RSA
cluster.link.auth.keystore.password=TESTE$%3030
cluster.link.auth.keystore.path=/opt/liferay/unicast/cluster_link_keystore.jks
cluster.link.auth.keystore.type=JKS
cluster.link.auth.value=TESTE$%3030

# Para ver terminal de logs identificando nó
web.server.display.node=true
```

### Observações
* Todos os passwords, aliases e caminhos do keystore **devem ser idênticos entre os nós**
* `web.server.display.node=true` ajuda a identificar qual nó está atendendo a requisição

---

## Passo 5 — Reiniciar os Nós do Liferay

Reinicie os containers do Liferay:

```bash
docker compose restart liferay-node1 liferay-node2
```

---

## Passo 6 — Validação

Verifique os logs do Liferay. Você deverá ver mensagens semelhantes a:

```text
Accepted view
```

✅ O Cluster Link agora está corretamente configurado para um ambiente Docker com Liferay.
