---
description: Atividade 3 — Criar novo Oracle Home a partir do Gold Image base e preparar para patching
---

# Atividade 3 — Criar Novo Oracle Home

## Contexto
Cria um segundo Oracle Home (`${ORACLE_HOME_19C_PATCHED}`) a partir do Gold Image base do 19c.
O banco continua rodando no home original (`${ORACLE_HOME_19C}`).
O OPatch e o patch RU serão aplicados neste novo home na Atividade 4.

## Pré-requisitos
- Atividade 2 concluída
- Em `/install`: `${GOLD_IMAGE_19C}`, `${OPATCH_ZIP}`, `${PATCH_19C_ZIP}`
- Espaço em `/u01`: ao menos 8 GB livres

## Carregar variáveis

Leia `.env` com o Read tool. Nunca exiba senhas.

```bash
SSH="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR root@${VM_IP}"
```

---

## FASE 1 — Verificar pré-requisitos

### 1.1 — Espaço e arquivos disponíveis

```bash
${SSH} "
echo '=== Espaco em /u01 ==='
df -h /u01

echo ''
echo '=== Arquivos em /install ==='
ls -lh /install/*.zip /install/*.jar 2>/dev/null
"
```

Confirmar:
- Ao menos 8 GB livres em `/u01`
- `${GOLD_IMAGE_19C}`, `${OPATCH_ZIP}` e `${PATCH_19C_ZIP}` presentes

---

## FASE 2 — Criar o novo Oracle Home

### 2.1 — Criar diretório e extrair Gold Image

```bash
${SSH} "
mkdir -p ${ORACLE_HOME_19C_PATCHED}
chown -R ${ORACLE_USER}:${ORACLE_GROUP} ${ORACLE_HOME_19C_PATCHED}
echo 'Diretorio criado:'
ls -la $(dirname ${ORACLE_HOME_19C_PATCHED})
"
```

```bash
${SSH} "su - ${ORACLE_USER} -c '
cd ${ORACLE_HOME_19C_PATCHED}
echo \"Extraindo ${GOLD_IMAGE_19C}...\"
unzip -q ${INSTALL_DIR}/${GOLD_IMAGE_19C}
echo \"Extracao concluida\"
ls ${ORACLE_HOME_19C_PATCHED} | head -5
' 2>&1"
```

### 2.2 — Executar instalação silent

```bash
${SSH} "su - ${ORACLE_USER} -c '
export CV_ASSUME_DISTID=OEL8.0
${ORACLE_HOME_19C_PATCHED}/runInstaller -silent -ignorePrereqFailure \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=${VM_HOSTNAME} \
  UNIX_GROUP_NAME=${ORACLE_GROUP} \
  INVENTORY_LOCATION=${ORACLE_INVENTORY} \
  ORACLE_HOME=${ORACLE_HOME_19C_PATCHED} \
  ORACLE_BASE=${ORACLE_BASE} \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=${DBA_GROUP} \
  oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba \
  oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba \
  oracle.install.db.OSRACDBA_GROUP=racdba \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
  DECLINE_SECURITY_UPDATES=true
' 2>&1 | tail -15"
```

**Nota**: `CV_ASSUME_DISTID=OEL8.0` é necessário para evitar NullPointerException do installer 19.3 no OL 8.x.

### 2.3 — Executar root.sh do novo home

```bash
${SSH} "${ORACLE_HOME_19C_PATCHED}/root.sh"
```

---

## FASE 3 — Verificar o novo Oracle Home

### 3.1 — Confirmar instalação

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/OPatch:\$PATH

echo \"=== Versao Oracle ===\"
\$ORACLE_HOME/bin/sqlplus -v

echo \"=== OPatch version ===\"
opatch version

echo \"=== Patches aplicados ===\"
opatch lsinventory 2>&1 | grep -E \"Oracle Database|Installed|Patch +[0-9]\"
'"
```

### 3.2 — Confirmar banco ainda no home original

```bash
${SSH} 'cat > /tmp/check_home.sql << '"'"'ENDSQL'"'"'
SELECT instance_name, version, status FROM v$instance;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C}"'; export ORACLE_SID='"${ORACLE_SID}"'; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_home.sql"'
```

---

## Resultado esperado

- `${ORACLE_HOME_19C_PATCHED}` criado com Oracle 19.3.0 (base, sem patch ainda)
- OPatch e patch RU prontos em `/install` para a Atividade 4
- Banco ORCL ainda rodando no home original `${ORACLE_HOME_19C}`

## Próximo passo

Atividade 4 — aplicar OPatch atualizado + patch `${PATCH_19C_ZIP}` no novo home usando AutoUpgrade.
