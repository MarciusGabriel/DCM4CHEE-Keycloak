
DCM4CHEE + Keycloak com Correções
=================================


Documentação completa para configurar o DCM4CHEE integrado ao Keycloak, com instruções passo a passo, resolução de erros e orientações para replicar o ambiente.




---


📁 Estrutura Inicial do Projeto
------------------------------


Antes de rodar o `docker-compose.yml`, crie uma pasta para o projeto e coloque dentro dela o arquivo `docker-compose.yml` e as pastas de volumes:



```
mkdir dcm4chee-arc && cd dcm4chee-arc
nano docker-compose.yml

```

Em seguida, crie as pastas obrigatórias:



```
mkdir -p ldap slapd.d mysql keycloak db wildfly storage
sudo chown -R 1000:1000 storage wildfly keycloak

```



---


🐳 Docker Compose
----------------


Use o seguinte `docker-compose.yml` como base (versões testadas e funcionais):



```
version: "3"
services:
  ldap:
    image: dcm4che/slapd-dcm4chee:2.6.7-33.1
    ports:
      - "389:389"
      - "636:636"
    volumes:
      - ./ldap:/var/lib/openldap/openldap-data
      - ./slapd.d:/etc/openldap/slapd.d

  mariadb:
    image: mariadb:10.11.4
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: keycloak
      MYSQL_USER: keycloak
      MYSQL_PASSWORD: keycloak
    volumes:
      - ./mysql:/var/lib/mysql

  keycloak:
    image: dcm4che/keycloak:26.0.6
    ports:
      - "8843:8843"
    environment:
      KC_HTTPS_PORT: 8843
      KC_HOSTNAME: https://localhost:8843
      KC_HOSTNAME_STRICT: 'false'
      KC_HOSTNAME_STRICT_HTTPS: 'false'
      KC_DB: mariadb
      KC_DB_URL_DATABASE: keycloak
      KC_DB_URL_HOST: mariadb
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak
      KC_METRICS_ENABLED: 'true'
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: changeit
    depends_on:
      - ldap
      - mariadb
    volumes:
      - ./keycloak:/opt/keycloak/data

  db:
    image: dcm4che/postgres-dcm4chee:17.1-33
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: pacsdb
      POSTGRES_USER: pacs
      POSTGRES_PASSWORD: pacs
    volumes:
      - ./db:/var/lib/postgresql/data

  arc:
    image: dcm4che/dcm4chee-arc-psql:5.33.1-secure
    ports:
      - "8080:8080"
      - "8443:8443"
      - "9990:9990"
      - "9993:9993"
      - "11112:11112"
      - "2762:2762"
      - "2575:2575"
      - "12575:12575"
    environment:
      POSTGRES_DB: pacsdb
      POSTGRES_USER: pacs
      POSTGRES_PASSWORD: pacs
      AUTH_SERVER_URL: https://keycloak:8843
      UI_AUTH_SERVER_URL: https://localhost:8843
      WILDFLY_CHOWN: /storage
      WILDFLY_WAIT_FOR: ldap:389 db:5432 keycloak:8843
    depends_on:
      - ldap
      - keycloak
      - db
    volumes:
      - ./wildfly:/opt/wildfly/standalone
      - ./storage:/storage

```
## 🔓 Portas Importantes

| Serviço        | Porta Externa                         | Descrição                    |
| -------------- | ------------------------------------- | ---------------------------- |
| Keycloak       | `8843`                                | Interface web segura (HTTPS) |
| DCM4CHEE Arc   | `8443`                                | Interface web do DCM4CHEE    |
| DICOM          | `11112`                               | Protocolo padrão DICOM       |
| Banco de dados | `5432` (PostgreSQL), `3306` (MariaDB) | Acesso interno               |

> 🔥 Libere todas essas portas no firewall se estiver usando servidor remoto.


---

## 🔑 Keycloak Realm Export

```bash
docker exec -it dcm4chee-arc_keycloak_1 /bin/bash
/opt/keycloak/bin/kc.sh export --dir /opt/keycloak/data/import --realm dcm4che --users realm_file
```

> O arquivo `realm-export.json` será salvo em `./keycloak/data/import`.

Importar novamente:

```bash
/opt/keycloak/bin/kc.sh start --import-realm
```

---

## 👤 Criar novo usuário admin

Acesse `https://localhost:8843` > Users > Create user  
Setar senha > Role Mappings > Adicionar:

- `manage-users`
- `manage-realm`
- `manage-clients`
- `query-users`
- `view-realm`

---

## ⚠ Possíveis Erros

| Erro | Solução |
|------|---------|
| Forbidden na UI2 | Verifique roles e client authentication |
| Could not resolve hostname | Edite `/etc/hosts` ou DNS |
| Missing id_token_hint | Gere logout com JWT válido |
| Truststore-type | Pode ser ignorado se não customizado |

---

## 🔄 Reset

```bash
docker-compose down -v --remove-orphans
sudo rm -rf ldap slapd.d mysql keycloak db wildfly storage
```

---

## 🔐 Logout URL

```
https://localhost:8843/realms/dcm4che/protocol/openid-connect/logout?redirect_uri=https://localhost:8443/dcm4chee-arc/ui2/
```

---

## 🛠️ Roles via terminal

```bash
docker exec -it dcm4chee-arc_keycloak_1 /opt/keycloak/bin/kc.sh create roles --realm dcm4che --name admin-full
kc.sh add-roles --realm dcm4che --username meuuser --roles admin-full

Para adicionar a um usuário:

kc.sh add-roles --realm dcm4che --username meuuser --roles admin-full
```

---

## 🗑️ Limpeza

```bash
docker system prune -af
docker system df
sudo rm -rf /var/lib/docker/volumes/<volume-na
```

---

## 📦 Subida de imagens separadas

```bash
docker pull dcm4che/keycloak:26.0.6
docker pull dcm4che/dcm4chee-arc-psql:5.33.1-secure
docker-compose up -d
```

---
