---
description: Atividade 2 — Baixar autoupgrade.jar mais recente via wget e verificar estado do Oracle Home 19c
---

# Atividade 2 — Download e Atualização do AutoUpgrade

## Contexto
O `autoupgrade.jar` copiado na Atividade 1 é a versão base do 19.3 (2019). Esta atividade:
1. Baixa a versão mais recente do `autoupgrade.jar` via wget (endpoint público Oracle)
2. Verifica os patches instalados no Oracle Home 19c

**Todos os arquivos `.cfg` do AutoUpgrade são criados pela IA com a sintaxe correta e salvos em `${INSTALL_DIR}`. O usuário não precisa criá-los.**

## Pré-requisitos
- Atividade 1 concluída (banco ORCL rodando, Java 17 instalado, autoupgrade.jar em `${INSTALL_DIR}`)

## Carregar variáveis

Leia `.env` com o Read tool e extraia as variáveis. Nunca exiba senhas.

Definir variáveis SSH e JAVA (usar em todos os comandos):
```bash
SSH="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR root@${VM_IP}"
# AutoUpgrade 26.x requer Java 11 — usar o JDK embutido no Oracle Home
JAVA="${ORACLE_HOME_19C}/jdk/bin/java"
```

---

## FASE 1 — Verificar Java

### 1.1 — Confirmar Java disponível

```bash
${SSH} "java -version 2>&1 && which java"
```

Se Java não estiver instalado:
```bash
${SSH} "dnf install -y java-17-openjdk 2>&1 | tail -3 && java -version"
```

**Nota**: Java 17 é instalado para uso geral do SO, mas o AutoUpgrade 26.x requer **Java 11**.
Usar sempre o Java embutido no Oracle Home (`${ORACLE_HOME_19C}/jdk/bin/java`), que é a versão 11 certificada pela Oracle.

---

## FASE 2 — Baixar autoupgrade.jar mais recente

### 2.1 — Fazer backup do jar atual

```bash
${SSH} "
cp ${AUTOUPGRADE_JAR} ${AUTOUPGRADE_JAR}.bak_\$(date +%Y%m%d)
ls -lh ${INSTALL_DIR}/autoupgrade*.jar*
"
```

### 2.2 — Instalar wget e baixar jar atualizado

```bash
${SSH} "
dnf install -y wget 2>&1 | tail -2
wget -O ${AUTOUPGRADE_JAR} https://download.oracle.com/otn-pub/otn_software/autoupgrade.jar
"
```

### 2.3 — Confirmar versão baixada

```bash
${SSH} "
file ${AUTOUPGRADE_JAR}
ls -lh ${AUTOUPGRADE_JAR}
${JAVA} -jar ${AUTOUPGRADE_JAR} -version 2>/dev/null | head -5
"
```

Resultado esperado: versão `26.x`, suporte a targets `12.2,18,19,21,23,26`.

**Nota (2026-08-13):** o Oracle Home 19c ainda na base (antes do patch da Atividade 4) traz
um JDK **8** embutido (`jdk/bin/java` → `1.8.0_201`), não Java 11 — o JDK 11 só chega junto
com o RU aplicado na Atividade 4. Mesmo assim, `-jar autoupgrade.jar -version` rodou sem
erro com esse JDK 8 nesta sessão (retornou `26.5.x` normalmente). O aviso do `CLAUDE.md`
sobre "Unsupported Java Runtime Environment" no Java 11 provavelmente só se manifesta em
`-mode analyze`/`-deploy` de verdade — não foi reproduzido aqui só com `-version`.

---

## FASE 3 — Verificar patches instalados no Oracle Home

### 3.1 — Listar patches via opatch

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C}
export PATH=\$ORACLE_HOME/OPatch:\$PATH
echo \"=== Patches no Oracle Home 19c ===\"
opatch lsinventory 2>&1 | grep -E \"Oracle Database|Patch +[0-9]|Applied on|Installed\"
'"
```

### 3.2 — Verificar versão completa e RUs via sqlplus

Usar arquivo temporário para evitar problemas de escaping do `$` via SSH:

```bash
${SSH} 'cat > /tmp/check_version.sql << '"'"'ENDSQL'"'"'
SELECT version_full FROM v$instance;
SELECT patch_id, status, description FROM dba_registry_sqlpatch ORDER BY action_time DESC FETCH FIRST 5 ROWS ONLY;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C}"'; export ORACLE_SID='"${ORACLE_SID}"'; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_version.sql"'
```

---

## Resultado esperado

- `autoupgrade.jar` em `${INSTALL_DIR}` na versão 26.x, compatível com Java 17
- Oracle Home 19c na versão base 19.3.0.0.0 com RU aplicado (sem patches adicionais ainda)

## Arquivos gerados nesta atividade

| Arquivo | Descrição |
|---|---|
| `${INSTALL_DIR}/autoupgrade.jar` | Versão mais recente (26.x) baixada via wget |
| `${INSTALL_DIR}/autoupgrade.jar.bak_YYYYMMDD` | Backup do jar original |
