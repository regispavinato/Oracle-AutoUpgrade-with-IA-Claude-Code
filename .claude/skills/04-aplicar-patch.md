---
description: Atividade 4 — Atualizar OPatch, aplicar RU no novo home e mover banco com AutoUpgrade
---

# Atividade 4 — Aplicar Patch com AutoUpgrade

## Contexto
Fluxo de patching out-of-place:
1. Atualizar OPatch no `${ORACLE_HOME_19C_PATCHED}`
2. Aplicar o RU (`${PATCH_19C_ZIP}`) no novo home via OPatch
3. Usar AutoUpgrade (`deploy`) para mover o banco ORCL para o home patchado

O banco fica no ar até o momento do deploy — AutoUpgrade cuida de parar, trocar o home, rodar datapatch/utlrp e reiniciar.

## Pré-requisitos
- Atividade 3 concluída: `${ORACLE_HOME_19C_PATCHED}` criado (base 19.3, sem patch)
- Em `/install`: `${OPATCH_ZIP}` e `${PATCH_19C_ZIP}`
- Banco ORCL rodando no home original

## Carregar variáveis

Leia `.env` com o Read tool. Nunca exiba senhas.

```bash
SSH="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 root@${VM_IP}"
# AutoUpgrade 26.x requer Java 11 — usar o JDK embutido no Oracle Home
JAVA="${ORACLE_HOME_19C}/jdk/bin/java"
```

---

## FASE 1 — Atualizar OPatch no novo home

### 1.1 — Substituir OPatch

```bash
${SSH} "
cd ${ORACLE_HOME_19C_PATCHED}
echo '=== OPatch atual ==='
${ORACLE_HOME_19C_PATCHED}/OPatch/opatch version

echo 'Substituindo OPatch...'
mv ${ORACLE_HOME_19C_PATCHED}/OPatch ${ORACLE_HOME_19C_PATCHED}/OPatch.bak
unzip -q ${INSTALL_DIR}/${OPATCH_ZIP} -d ${ORACLE_HOME_19C_PATCHED}
chown -R ${ORACLE_USER}:${ORACLE_GROUP} ${ORACLE_HOME_19C_PATCHED}/OPatch

echo '=== OPatch atualizado ==='
${ORACLE_HOME_19C_PATCHED}/OPatch/opatch version
"
```

---

## FASE 2 — Aplicar RU no novo home via OPatch

### 2.1 — Extrair e aplicar patch

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/OPatch:\$PATH

PATCH_NUM=\$(unzip -Z1 ${INSTALL_DIR}/${PATCH_19C_ZIP} | head -1 | cut -d/ -f1)
echo \"Patch number: \$PATCH_NUM\"

mkdir -p /tmp/patch_apply
cd /tmp/patch_apply
unzip -oq ${INSTALL_DIR}/${PATCH_19C_ZIP}

echo \"Aplicando patch \$PATCH_NUM em ${ORACLE_HOME_19C_PATCHED}...\"
cd /tmp/patch_apply/\$PATCH_NUM
opatch apply -silent
echo \"OPatch apply concluido\"
' 2>&1 | tail -20"
```

### 2.2 — Confirmar patch aplicado

**Nota (2026-08-13):** o padrão `"Patch [0-9]"` (um espaço) não bate com a saída real do
OPatch 12.2.0.1.51, que formata como `Patch  39034528` (dois espaços) — o patch tinha sido
aplicado com sucesso, mas esse grep não mostrava nada, sugerindo falsamente que não achou
nada. Usar `"Patch +[0-9]"` (um ou mais espaços):

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/OPatch:\$PATH
opatch lsinventory 2>&1 | grep -E \"Oracle Database|Patch +[0-9]|Applied on\"
'"
```

---

## FASE 3 — Mover banco para o home patchado com AutoUpgrade

### 3.1 — Verificar estado do banco

```bash
${SSH} 'cat > /tmp/check_db.sql << '"'"'ENDSQL'"'"'
SELECT name, open_mode FROM v$database;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C}"'; export ORACLE_SID='"${ORACLE_SID}"'; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_db.sql"'
```

### 3.2 — Gerar autoupgrade_patch.cfg

Exibir para o usuário verificar antes de criar:

```
global.autoupg_log_dir=${AUTOUPGRADE_LOG_DIR}

upg1.source_home=${ORACLE_HOME_19C}
upg1.target_home=${ORACLE_HOME_19C_PATCHED}
upg1.sid=${ORACLE_SID}
upg1.log_dir=${AUTOUPGRADE_LOG_DIR}/${ORACLE_SID}
upg1.upgrade_node=${VM_HOSTNAME}
upg1.run_utlrp=yes
upg1.timezone_upg=yes
```

```bash
${SSH} "
mkdir -p ${AUTOUPGRADE_LOG_DIR}
chown ${ORACLE_USER}:${ORACLE_GROUP} ${AUTOUPGRADE_LOG_DIR}

cat > ${INSTALL_DIR}/autoupgrade_patch.cfg << 'EOF'
global.autoupg_log_dir=${AUTOUPGRADE_LOG_DIR}

upg1.source_home=${ORACLE_HOME_19C}
upg1.target_home=${ORACLE_HOME_19C_PATCHED}
upg1.sid=${ORACLE_SID}
upg1.log_dir=${AUTOUPGRADE_LOG_DIR}/${ORACLE_SID}
upg1.upgrade_node=${VM_HOSTNAME}
upg1.run_utlrp=yes
upg1.timezone_upg=yes
EOF
chown ${ORACLE_USER}:${ORACLE_GROUP} ${INSTALL_DIR}/autoupgrade_patch.cfg
cat ${INSTALL_DIR}/autoupgrade_patch.cfg
"
```

### 3.3 — Executar analyze

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C}
export ORACLE_SID=${ORACLE_SID}
export PATH=\$ORACLE_HOME/bin:\$PATH

${JAVA} -jar ${AUTOUPGRADE_JAR} \
  -config ${INSTALL_DIR}/autoupgrade_patch.cfg \
  -mode analyze \
  -noconsole
' 2>&1"
```

Se houver `CHECK_FAILED` crítico, avaliar com o usuário antes de prosseguir.

### 3.4 — Executar deploy

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C}
export ORACLE_SID=${ORACLE_SID}
export PATH=\$ORACLE_HOME/bin:\$PATH

${JAVA} -jar ${AUTOUPGRADE_JAR} \
  -config ${INSTALL_DIR}/autoupgrade_patch.cfg \
  -mode deploy \
  -noconsole
' 2>&1"
```

**Duração estimada**: 10–20 minutos. O AutoUpgrade executa automaticamente:
1. Fixups pré-patch
2. Shutdown do banco
3. Troca do Oracle Home no oratab
4. Startup no novo home
5. datapatch + utlrp
6. Validação pós-patch

---

## FASE 4 — Verificação pós-patch

### 4.1 — Verificar Oracle Home ativo e oratab

```bash
${SSH} "
echo '=== /etc/oratab ==='
grep ${ORACLE_SID} /etc/oratab
"
```

### 4.2 — Verificar versão e patches no banco

```bash
${SSH} 'cat > /tmp/check_patch.sql << '"'"'ENDSQL'"'"'
SELECT version_full FROM v$instance;
SELECT patch_id, status, description FROM dba_registry_sqlpatch ORDER BY action_time DESC FETCH FIRST 5 ROWS ONLY;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C_PATCHED}"'; export ORACLE_SID='"${ORACLE_SID}"'; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_patch.sql"'
```

### 4.3 — Verificar listener

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl status
'"
```

### 4.4 — Migrar listener para o novo Oracle Home

O AutoUpgrade não migra o listener automaticamente. O listener continua usando o binário e o
`listener.ora` do home antigo e precisa ser recriado no novo home.

```bash
${SSH} "su - ${ORACLE_USER} -c '
echo \"=== Parando listener no home antigo ===\"
export ORACLE_HOME=${ORACLE_HOME_19C}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl stop

echo \"=== Copiando listener.ora e tnsnames.ora para novo home ===\"
cp ${ORACLE_HOME_19C}/network/admin/listener.ora ${ORACLE_HOME_19C_PATCHED}/network/admin/listener.ora
cp ${ORACLE_HOME_19C}/network/admin/tnsnames.ora ${ORACLE_HOME_19C_PATCHED}/network/admin/tnsnames.ora 2>/dev/null || true

echo \"=== Iniciando listener no novo home ===\"
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl start
'"
```

Forçar registro dinâmico do banco no novo listener:

```bash
${SSH} 'cat > /tmp/register.sql << '"'"'ENDSQL'"'"'
ALTER SYSTEM REGISTER;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C_PATCHED}"'; export ORACLE_SID='"${ORACLE_SID}"'; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/register.sql"'
```

Verificar serviços registrados:

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl status 2>&1 | grep -E \"Parameter File|Version TNSLSNR|Services Summary|Service|Instance\"
'"
```

Resultado esperado: `Listener Parameter File` apontando para `${ORACLE_HOME_19C_PATCHED}` e serviços ORCL/ORCLXDB com status `READY`.

### 4.5 — Atualizar serviços systemd

```bash
${SSH} "
sed -i 's|${ORACLE_HOME_19C}|${ORACLE_HOME_19C_PATCHED}|g' /etc/systemd/system/oracle-listener.service
sed -i 's|${ORACLE_HOME_19C}|${ORACLE_HOME_19C_PATCHED}|g' /etc/systemd/system/oracle-database.service
systemctl daemon-reload
systemctl status oracle-database.service --no-pager | head -5
"
```

---

## Resultado esperado

- `${ORACLE_HOME_19C_PATCHED}` com RU aplicado (`dba_registry_sqlpatch` com status `SUCCESS`)
- Banco ORCL rodando em `${ORACLE_HOME_19C_PATCHED}`
- `/etc/oratab` apontando para o novo home
- Listener funcionando no novo home
- Serviços systemd atualizados
