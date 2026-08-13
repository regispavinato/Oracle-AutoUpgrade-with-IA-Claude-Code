---
description: Atividade 5 — Converter banco 19c non-CDB para PDB em novo CDB usando AutoUpgrade (noncdbtopdb)
---

# Atividade 5 — Converter non-CDB para Multitenant com AutoUpgrade

## Contexto
O AutoUpgrade 26.x converte um banco non-CDB em PDB usando o modo `noncdbtopdb`. A versão 26.x
**exige** um CDB alvo já existente — o parâmetro `target_cdb=NEW` não é mais suportado.

> **TRANSIÇÃO DE SID:**
> - **Antes desta atividade**: banco `ORCL` (non-CDB), `ORACLE_SID=ORCL` — valor correto do `.env`
> - **Após esta atividade**: banco `CDBORCL` (CDB) com PDB `ORCLPDB`, `ORACLE_SID=CDBORCL`
> - O `.env` continua com `ORACLE_SID=ORCL` — nas skills **06, 07 e 08** ignorar o valor do `.env` e usar `CDBORCL`

Fluxo adotado (REAPROVEITANDO o `dbhome_1` já patchado — não cria home novo):
1. ~~Criar dbhome_2~~ → **REAPROVEITAR `${ORACLE_HOME_19C_PATCHED}` (dbhome_1)**, que já está em 19.31 com a RU aplicada na Atividade 4. Não é necessário criar nem patchear um novo home.
2. Criar CDB vazio `CDBORCL` com DBCA silent **no próprio `dbhome_1`** (o CDB e o non-CDB ORCL coexistem no mesmo home, com SIDs e datafiles distintos)
3. Listener já está no `dbhome_1` (migrado na Atividade 4) — apenas registrar `CDBORCL`
4. AutoUpgrade `noncdbtopdb` com **`source_home = target_home = dbhome_1`**: ORCL (non-CDB) → ORCLPDB (PDB em `CDBORCL`). Conversão 19.31→19.31 sem upgrade de versão, então o mesmo home é válido.
5. Verificar e remover **apenas o `db_1`** (home base original, sem patch). O `dbhome_1` passa a ser o home único.
6. Atualizar `.bash_profile` do oracle, systemd e oratab para `CDBORCL` + `dbhome_1`

> **Nota de ajuste (2026-07-11):** a versão original desta skill criava um `dbhome_2` fresh e o patcheava. Como o `dbhome_1` da Atividade 4 já está no nível de RU correto, reaproveitá-lo elimina ~20 min de install+patch e a migração de listener, sem perda técnica (a conversão não muda de versão).

## Pré-requisitos
- Atividade 4 concluída: banco ORCL rodando em `${ORACLE_HOME_19C_PATCHED}` (dbhome_1), patchado com RU p39034528
- `${GOLD_IMAGE_19C}` disponível em `/install`
- `${OPATCH_ZIP}` e `${PATCH_19C_ZIP}` disponíveis em `/install`
- `autoupgrade.jar` 26.x em `${AUTOUPGRADE_JAR}`
- Java 17 instalado (`java-17-openjdk`)
- Espaço em `/u01`: ao menos 8 GB livres para `dbhome_2`

## Carregar variáveis

Leia `.env` com o Read tool. Nunca exiba senhas.

Variáveis adicionais desta atividade:
```
ORACLE_HOME_CDB=${ORACLE_HOME_19C_PATCHED}   # dbhome_1 reaproveitado (NÃO cria dbhome_2)
CDB_SID=CDBORCL
PDB_NAME=ORCLPDB
```

```bash
SSH="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 root@${VM_IP}"
# AutoUpgrade 26.x requer Java 11 — o JDK do Oracle Home 19.3 é Java 8, NÃO usar.
# Usar sempre o Java 11 instalado no SO (ver CLAUDE.md / feedback-java11-autoupgrade):
JAVA11=/usr/lib/jvm/java-11-openjdk/bin/java
```

---

## FASE 1 — ~~Criar novo Oracle Home~~ (PULAR — reaproveitar dbhome_1)

> **PULAR ESTA FASE.** O `${ORACLE_HOME_19C_PATCHED}` (dbhome_1) já existe, em 19.31 com a RU aplicada (Atividade 3+4). Não crie um home novo. Os blocos abaixo ficam apenas como referência histórica.

### 1.1 — Verificar espaço e criar diretório

```bash
${SSH} "
echo '=== Espaço em /u01 ==='
df -h /u01
mkdir -p ${ORACLE_HOME_19C_PATCHED}
chown -R ${ORACLE_USER}:${ORACLE_GROUP} ${ORACLE_HOME_19C_PATCHED}
"
```

### 1.2 — Extrair Gold Image em dbhome_2

```bash
${SSH} "su - ${ORACLE_USER} -c '
cd ${ORACLE_HOME_19C_PATCHED}
echo \"Extraindo ${GOLD_IMAGE_19C}...\"
unzip -q ${INSTALL_DIR}/${GOLD_IMAGE_19C}
echo \"Extração concluída\"
ls ${ORACLE_HOME_19C_PATCHED} | head -5
' 2>&1"
```

### 1.3 — Instalação silent

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

**Nota**: `CV_ASSUME_DISTID=OEL8.0` é obrigatório para o installer 19.3 no OL 8.x.

### 1.4 — Executar root.sh

```bash
${SSH} "${ORACLE_HOME_19C_PATCHED}/root.sh"
```

---

## FASE 2 — ~~Aplicar OPatch + RU~~ (PULAR — dbhome_1 já patchado)

> **PULAR ESTA FASE.** O `${ORACLE_HOME_19C_PATCHED}` já está com OPatch 12.2.0.1.51 e a RU 19.31 (p39034528) aplicados na Atividade 4. Blocos abaixo apenas para referência.

O ${ORACLE_HOME_19C_PATCHED} precisa estar no mesmo nível de RU que o banco fonte (ORCL).

### 2.1 — Substituir OPatch

```bash
${SSH} "
mv ${ORACLE_HOME_19C_PATCHED}/OPatch ${ORACLE_HOME_19C_PATCHED}/OPatch.bak
unzip -q ${INSTALL_DIR}/${OPATCH_ZIP} -d ${ORACLE_HOME_19C_PATCHED}
chown -R ${ORACLE_USER}:${ORACLE_GROUP} ${ORACLE_HOME_19C_PATCHED}/OPatch
${ORACLE_HOME_19C_PATCHED}/OPatch/opatch version
"
```

### 2.2 — Aplicar RU (opatch apply -silent)

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/OPatch:\$PATH

PATCH_NUM=\$(unzip -Z1 ${INSTALL_DIR}/${PATCH_19C_ZIP} | head -1 | cut -d/ -f1)
mkdir -p /tmp/patch_apply2
cd /tmp/patch_apply2
unzip -oq ${INSTALL_DIR}/${PATCH_19C_ZIP}
cd /tmp/patch_apply2/\$PATCH_NUM
opatch apply -silent
echo \"OPatch apply concluído\"
' 2>&1 | tail -10"
```

### 2.3 — Confirmar patch

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/OPatch:\$PATH
opatch lsinventory 2>&1 | grep -E \"Oracle Database|Patch +[0-9]|Applied on\"
'"
```

---

## FASE 3 — Criar CDB vazio com DBCA

O AutoUpgrade 26.x exige um CDB existente como `target_cdb` para `noncdbtopdb`.

> **PADRÃO OBRIGATÓRIO (lição da Atividade 5, ver CLAUDE.md § Regras Operacionais):** em VM com RAM restrita, **parar o `ORCL` ANTES de criar o CDB** — não como correção reativa, mas como passo 3.0 padrão. Rodar dois bancos simultâneos durante o DBCA pode causar swap pesado e fazer o `utlrp` do DBCA falhar sozinho (o DBCA então desfaz o CDB inteiro, o que parece um bug mas é apenas o rollback dele). Religar o `ORCL` só depois do DBCA confirmar "Database creation complete".

### 3.0 — Parar ORCL para liberar RAM (padrão, não reativo)

```bash
${SSH} 'cat > /tmp/stop_orcl.sql << '"'"'ENDSQL'"'"'
shutdown immediate
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C_PATCHED}"'; export ORACLE_SID=${ORACLE_SID}; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/stop_orcl.sql"'
```

Confirmar memória livre suficiente antes de prosseguir:
```bash
${SSH} "free -h | head -2"
```

### 3.1 — Criar CDB CDBORCL (sem PDBs)

**IMPORTANTE**: rodar DESTACADO com `nohup` (o DBCA pode levar 15-30 min e o shell tem timeout de ~10 min). Monitorar via `pgrep -f 'dbca -silent'` em vez de aguardar o retorno do SSH em foreground. **Nunca interromper/bounce a instância enquanto o processo `dbca` ainda estiver rodando** — se ele falhar sozinho, faz seu próprio rollback e apaga os arquivos do CDB.

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

nohup dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname CDBORCL \
  -sid CDBORCL \
  -createAsContainerDatabase true \
  -numberOfPDBs 0 \
  -characterSet AL32UTF8 \
  -nationalCharacterSet AL16UTF16 \
  -sysPassword ${ORACLE_PWD} \
  -systemPassword ${ORACLE_PWD} \
  -datafileDestination ${ORADATA_DIR}/CDBORCL \
  -recoveryAreaDestination ${FRA_DIR} \
  -recoveryAreaSize 2048 \
  -totalMemory 2048 \
  -redoLogFileSize 50 \
  -databaseType MULTIPURPOSE \
  -emConfiguration NONE \
  -enableArchive false \
  -ignorePreReqs > /tmp/dbca_cdborcl.out 2>&1 &
echo \"DBCA_PID=\$!\"
'"
```

Aguardar o processo `dbca` encerrar sozinho (poll a cada ~20-30s):
```bash
${SSH} "pgrep -f 'dbca -silent' >/dev/null && echo RODANDO || echo ENCERRADO"
```

Ao encerrar, confirmar sucesso ANTES de prosseguir (procurar "Database creation complete" e ausência de `FATAL`):
```bash
${SSH} "grep -iE 'Database creation complete|FATAL|Error while executing' /tmp/dbca_cdborcl.out /u01/app/oracle/cfgtoollogs/dbca/CDBORCL/CDBORCL.log 2>/dev/null | tail -6"
```

Religar o ORCL depois que o CDB estiver confirmado:
```bash
${SSH} 'cat > /tmp/start_orcl.sql << '"'"'ENDSQL'"'"'
startup
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C_PATCHED}"'; export ORACLE_SID=${ORACLE_SID}; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/start_orcl.sql"'
```

### 3.2 — Verificar CDB criado

```bash
${SSH} 'cat > /tmp/check_cdb.sql << '"'"'ENDSQL'"'"'
SELECT name, cdb, open_mode FROM v$database;
SELECT con_id, name, open_mode FROM v$pdbs ORDER BY con_id;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_cdb.sql"'
```

Esperado: `CDB=YES`, apenas PDB$SEED com `MOUNTED` ou `READ ONLY`.

---

## FASE 4 — Listener (já no dbhome_1 — apenas registrar CDBORCL)

> **AJUSTE:** o listener já foi migrado para o `dbhome_1` na Atividade 4. Como o CDB alvo também está no `dbhome_1`, não há migração a fazer — apenas registrar o `CDBORCL` (ver 4.2). Os comandos de parar/copiar/iniciar abaixo tornam-se no-op (mesmo home) e ficam como referência.

O listener deve estar no mesmo home que o CDB alvo antes do AutoUpgrade.

### 4.1 — Parar listener no home antigo e iniciar no dbhome_2

```bash
${SSH} "su - ${ORACLE_USER} -c '
echo \"=== Parando listener no home antigo ===\"
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl stop

echo \"=== Copiando listener.ora para dbhome_2 ===\"
cp ${ORACLE_HOME_19C_PATCHED}/network/admin/listener.ora ${ORACLE_HOME_19C_PATCHED}/network/admin/listener.ora
cp ${ORACLE_HOME_19C_PATCHED}/network/admin/tnsnames.ora ${ORACLE_HOME_19C_PATCHED}/network/admin/tnsnames.ora 2>/dev/null || true

echo \"=== Iniciando listener no dbhome_2 ===\"
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl start
'"
```

### 4.2 — Registrar bancos no novo listener

```bash
${SSH} 'cat > /tmp/register2.sql << '"'"'ENDSQL'"'"'
ALTER SYSTEM REGISTER;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/register2.sql"
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C_PATCHED}"'; export ORACLE_SID='"${ORACLE_SID}"'; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/register2.sql"'
```

---

## FASE 5 — AutoUpgrade: conversão noncdbtopdb

### 5.1 — Limpeza preventiva de jobs anteriores do AutoUpgrade (padrão, sempre executar)

> **PADRÃO OBRIGATÓRIO** (ver CLAUDE.md § Regras Operacionais): antes de QUALQUER `-mode analyze`/`-mode deploy`, descobrir e limpar o job mais recente do mesmo SID — não esperar o erro "unfinished execution" aparecer para só então reagir. Jobs de atividades anteriores (mesmo com `.cfg` diferente) ficam indexados pelo mesmo SID e bloqueiam a nova execução.

Descobrir o job mais recente para o SID e limpar (best-effort — "not found" é inofensivo):

```bash
${SSH} "
LAST_JOB=\$(ls -td ${AUTOUPGRADE_LOG_DIR}/${ORACLE_SID}/${ORACLE_SID}/*/ 2>/dev/null | head -1 | xargs -n1 basename 2>/dev/null)
echo \"Ultimo job encontrado para SID ${ORACLE_SID}: \${LAST_JOB:-nenhum}\"
if [ -n \"\$LAST_JOB\" ]; then
  su - ${ORACLE_USER} -c \"
  export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
  export ORACLE_SID=${ORACLE_SID}
  export PATH=\\\$ORACLE_HOME/bin:\\\$PATH
  ${JAVA11} -jar ${AUTOUPGRADE_JAR} -config ${INSTALL_DIR}/autoupgrade_convert.cfg -clear_recovery_data -jobs \$LAST_JOB
  \" 2>&1 | tail -5
fi
"
```

### 5.2 — Gerar autoupgrade_convert.cfg

Exibir para o usuário verificar antes de criar:

```
global.global_log_dir=${AUTOUPGRADE_LOG_DIR}

upg1.source_home=${ORACLE_HOME_19C_PATCHED}
upg1.target_home=${ORACLE_HOME_19C_PATCHED}
upg1.sid=${ORACLE_SID}
upg1.target_cdb=CDBORCL
upg1.target_pdb_name=ORCLPDB
upg1.log_dir=${AUTOUPGRADE_LOG_DIR}/${ORACLE_SID}_CONVERT
upg1.upgrade_node=${VM_HOSTNAME}
upg1.run_utlrp=yes
upg1.timezone_upg=yes
```

**Nota**: AutoUpgrade 26.x usa `global.global_log_dir` (não `global.autoupg_log_dir`).

```bash
${SSH} "
mkdir -p ${AUTOUPGRADE_LOG_DIR}
chown ${ORACLE_USER}:${ORACLE_GROUP} ${AUTOUPGRADE_LOG_DIR}

cat > ${INSTALL_DIR}/autoupgrade_convert.cfg << 'EOF'
global.global_log_dir=${AUTOUPGRADE_LOG_DIR}

upg1.source_home=${ORACLE_HOME_19C_PATCHED}
upg1.target_home=${ORACLE_HOME_19C_PATCHED}
upg1.sid=${ORACLE_SID}
upg1.target_cdb=CDBORCL
upg1.target_pdb_name=ORCLPDB
upg1.log_dir=${AUTOUPGRADE_LOG_DIR}/${ORACLE_SID}_CONVERT
upg1.upgrade_node=${VM_HOSTNAME}
upg1.run_utlrp=yes
upg1.timezone_upg=yes
EOF
chown ${ORACLE_USER}:${ORACLE_GROUP} ${INSTALL_DIR}/autoupgrade_convert.cfg
cat ${INSTALL_DIR}/autoupgrade_convert.cfg
"
```

### 5.3 — Executar analyze

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export ORACLE_SID=${ORACLE_SID}
export PATH=\$ORACLE_HOME/bin:\$PATH

${JAVA11} -jar ${AUTOUPGRADE_JAR} \
  -config ${INSTALL_DIR}/autoupgrade_convert.cfg \
  -mode analyze \
  -noconsole
' 2>&1"
```

### 5.4 — Executar deploy

**IMPORTANTE**: rodar DESTACADO com `nohup` (20-40 min — excede o timeout de shell). Monitorar via `pgrep -f 'autoupgrade.jar -config'` em vez de aguardar o retorno em foreground.

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export ORACLE_SID=${ORACLE_SID}
export PATH=\$ORACLE_HOME/bin:\$PATH

nohup ${JAVA11} -jar ${AUTOUPGRADE_JAR} \
  -config ${INSTALL_DIR}/autoupgrade_convert.cfg \
  -mode deploy \
  -noconsole > /tmp/autoupgrade_convert.out 2>&1 &
echo \"PID=\$!\"
'"
```

**Duração estimada**: 20-40 minutos. O AutoUpgrade executa as etapas `NONCDBTOPDB`:
1. PREFIXUPS — prepara o non-CDB
2. DRAIN — desliga o banco fonte (ORCL)
3. NONCDBTOPDB — conecta ao CDBORCL e executa `noncdb_to_pdb.sql`
4. POSTFIXUPS — ajustes pós-conversão
5. POSTUPGRADE — valida e abre ORCLPDB

---

## FASE 6 — Verificação pós-conversão

### 6.1 — Verificar CDB e PDB

```bash
${SSH} 'cat > /tmp/check_conv.sql << '"'"'ENDSQL'"'"'
SELECT name, cdb, open_mode, version_full FROM v$database;
SELECT con_id, name, open_mode, restricted FROM v$pdbs ORDER BY con_id;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_conv.sql"'
```

Esperado: `CDB=YES`, PDB `ORCLPDB` com `OPEN_MODE=READ WRITE`.

### 6.2 — Verificar patch no dicionário

```bash
${SSH} 'cat > /tmp/check_patch2.sql << '"'"'ENDSQL'"'"'
SELECT patch_id, status, description FROM dba_registry_sqlpatch ORDER BY action_time DESC FETCH FIRST 5 ROWS ONLY;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_patch2.sql"'
```

### 6.3 — Verificar listener

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl status 2>&1 | grep -E \"Parameter File|Version|Services|Service|Instance\"
'"
```

### 6.4 — Abrir ORCLPDB e persistir estado (padrão obrigatório, sempre em conjunto)

> **PADRÃO OBRIGATÓRIO** (ver CLAUDE.md § Regras Operacionais): após a conversão, a PDB normalmente fica `MOUNTED`. Executar SEMPRE as duas ações juntas, no mesmo passo — nunca só o `OPEN` sem o `SAVE STATE`, senão a PDB volta a `MOUNTED` no próximo restart do banco (o `dbstart`/systemd não abre PDBs sozinho).

```bash
${SSH} 'cat > /tmp/openpdb.sql << '"'"'ENDSQL'"'"'
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
SELECT con_id, name, open_mode FROM v$pdbs ORDER BY con_id;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/openpdb.sql"'
```

Registrar a PDB no listener em seguida:
```bash
${SSH} 'cat > /tmp/reg3.sql << '"'"'ENDSQL'"'"'
ALTER SYSTEM REGISTER;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/reg3.sql"'
```

### 6.5 — Ativar OMF e mover datafiles da PDB para dentro do diretório do CDB (padrão obrigatório)

> **PADRÃO OBRIGATÓRIO** (ver CLAUDE.md § Regras Operacionais, descoberto em 2026-07-19). A conversão `noncdbtopdb` **não move os datafiles** — a PDB resultante continua fisicamente em `${ORADATA_DIR}/${ORACLE_SID}/*.dbf` (o caminho do non-CDB original), fora da estrutura `${ORADATA_DIR}/CDBORCL/`. Isso deixa os arquivos da PDB num diretório com nome do SID antigo, inconsistente com o padrão já usado pela `pdbseed` (subdiretório `${ORADATA_DIR}/CDBORCL/pdbseed/`). Sempre ativar OMF (`db_create_file_dest`) e mover os datafiles/tempfile da PDB para `${ORADATA_DIR}/CDBORCL/ORCLPDB/` logo após a conversão — a operação é **online** (`ALTER DATABASE MOVE DATAFILE`, banco continua `OPEN`).

Ativar OMF no CDB\$ROOT e na PDB (define o destino padrão para qualquer arquivo criado dali em diante — novas tablespaces, novas PDBs, etc.):
```bash
${SSH} 'cat > /tmp/enable_omf.sql << '"'"'ENDSQL'"'"'
ALTER SESSION SET CONTAINER=CDB$ROOT;
ALTER SYSTEM SET db_create_file_dest='"'"'${ORADATA_DIR}/CDBORCL'"'"' SCOPE=BOTH;
ALTER SESSION SET CONTAINER=ORCLPDB;
ALTER SYSTEM SET db_create_file_dest='"'"'${ORADATA_DIR}/CDBORCL'"'"' SCOPE=BOTH;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/enable_omf.sql"'
```

Criar o diretório destino e mover os 4 datafiles (online):
```bash
${SSH} "mkdir -p ${ORADATA_DIR}/CDBORCL/ORCLPDB; chown ${ORACLE_USER}:${ORACLE_GROUP} ${ORADATA_DIR}/CDBORCL/ORCLPDB"

${SSH} 'cat > /tmp/move_pdb_files.sql << '"'"'ENDSQL'"'"'
ALTER SESSION SET CONTAINER=ORCLPDB;
ALTER DATABASE MOVE DATAFILE '"'"'${ORADATA_DIR}/${ORACLE_SID}/system01.dbf'"'"' TO '"'"'${ORADATA_DIR}/CDBORCL/ORCLPDB/system01.dbf'"'"';
ALTER DATABASE MOVE DATAFILE '"'"'${ORADATA_DIR}/${ORACLE_SID}/sysaux01.dbf'"'"' TO '"'"'${ORADATA_DIR}/CDBORCL/ORCLPDB/sysaux01.dbf'"'"';
ALTER DATABASE MOVE DATAFILE '"'"'${ORADATA_DIR}/${ORACLE_SID}/undotbs01.dbf'"'"' TO '"'"'${ORADATA_DIR}/CDBORCL/ORCLPDB/undotbs01.dbf'"'"';
ALTER DATABASE MOVE DATAFILE '"'"'${ORADATA_DIR}/${ORACLE_SID}/users01.dbf'"'"' TO '"'"'${ORADATA_DIR}/CDBORCL/ORCLPDB/users01.dbf'"'"';
SELECT file#, name FROM v$datafile ORDER BY 1;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/move_pdb_files.sql"'
```

Mover o tempfile — `ALTER DATABASE MOVE DATAFILE` **não suporta tempfiles** (retorna `ORA-01516`); usar add+drop:
```bash
${SSH} 'cat > /tmp/move_pdb_temp.sql << '"'"'ENDSQL'"'"'
ALTER SESSION SET CONTAINER=ORCLPDB;
ALTER TABLESPACE TEMP ADD TEMPFILE '"'"'${ORADATA_DIR}/CDBORCL/ORCLPDB/temp01.dbf'"'"' SIZE 250M REUSE AUTOEXTEND ON;
ALTER DATABASE TEMPFILE '"'"'${ORADATA_DIR}/${ORACLE_SID}/temp01.dbf'"'"' DROP INCLUDING DATAFILES;
SELECT file#, name FROM v$tempfile ORDER BY 1;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/move_pdb_temp.sql"'
```

Confirmar que nada mais referencia o diretório antigo antes de removê-lo:
```bash
${SSH} 'cat > /tmp/check_old_files.sql << '"'"'ENDSQL'"'"'
SELECT name FROM v$datafile
UNION ALL SELECT name FROM v$tempfile
UNION ALL SELECT member FROM v$logfile
UNION ALL SELECT name FROM v$controlfile
ORDER BY 1;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_old_files.sql"'
```

Se nenhuma linha do resultado começar com `${ORADATA_DIR}/${ORACLE_SID}/`, o que resta no diretório é só o control file e os redo logs do non-CDB original (nunca usados pelo CDB, artefatos órfãos da criação do ORCL na Atividade 1) — remover:
```bash
${SSH} "rm -rf ${ORADATA_DIR}/${ORACLE_SID}"
```

---

## FASE 7 — Remover Oracle Homes antigos

Após verificar que tudo funciona em `${ORACLE_HOME_19C_PATCHED}` (dbhome_1), remover o `db_1` (home base sem patch) para liberar espaço.

### 7.1 — Confirmar que ORCL não está mais rodando

```bash
${SSH} "ps aux | grep pmon | grep -v grep"
```

Deve mostrar apenas `ora_pmon_CDBORCL`.

### 7.2 — Remover APENAS db_1 (dbhome_1 é o home que fica)

> **AJUSTE:** como reaproveitamos o `dbhome_1` como home único do CDBORCL, removemos **somente** o `db_1` (home base original, sem patch). NÃO remover o `dbhome_1`. Antes de remover, fazer detach do `db_1` no inventário.

```bash
${SSH} "
echo '=== Espaço antes ==='
df -h /u01

echo 'Detach db_1 do inventario...'
su - ${ORACLE_USER} -c '/u01/app/oracle/product/19.3.0/db_1/oui/bin/runInstaller -detachHome -silent ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1' 2>&1 | tail -3

echo 'Removendo db_1...'
rm -rf /u01/app/oracle/product/19.3.0/db_1

echo '=== Espaço depois ==='
df -h /u01
"
```

---

## FASE 8 — Atualizar configurações do sistema

### 8.1 — Atualizar .bash_profile do oracle

O `.bash_profile` do usuário oracle precisa apontar para o novo home e SID.
**Após esta atividade o SID passa a ser CDBORCL — atualizar obrigatoriamente.**

```bash
${SSH} "
# Substituir in-place (nunca deletar+anexar — isso move as linhas para o fim do
# arquivo e desalinha qualquer bloco que dependa da ordem, ex. oraenv)
sed -i 's|^export ORACLE_HOME=.*|export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}|' /home/oracle/.bash_profile
sed -i 's|^export ORACLE_SID=.*|export ORACLE_SID=CDBORCL|' /home/oracle/.bash_profile

echo '=== .bash_profile atualizado ==='
grep -E 'ORACLE_HOME|ORACLE_SID|PATH' /home/oracle/.bash_profile | grep -v '^#'
"
```

Verificar que as variáveis ficam corretas ao logar:
```bash
${SSH} "su - oracle -c 'echo ORACLE_HOME=\$ORACLE_HOME; echo ORACLE_SID=\$ORACLE_SID'"
```

### 8.2 — Atualizar /etc/oratab

```bash
${SSH} "
# Remover entradas antigas de ORCL e CDBORCL e adicionar nova entrada
grep -v '^ORCL:\|^CDBORCL:' /etc/oratab > /tmp/oratab.new || true
echo 'CDBORCL:${ORACLE_HOME_19C_PATCHED}:Y' >> /tmp/oratab.new
cp /tmp/oratab.new /etc/oratab
echo '=== /etc/oratab ==='
grep -v '^#' /etc/oratab | grep -v '^$'
"
```

### 8.3 — Atualizar serviços systemd

```bash
${SSH} "
for svc in oracle-database.service oracle-listener.service; do
  if [ -f /etc/systemd/system/\$svc ]; then
    sed -i 's|/19.3.0/db_1|/19.3.0/dbhome_1|g; s|ORACLE_SID=ORCL$|ORACLE_SID=CDBORCL|g' /etc/systemd/system/\$svc
  fi
done
systemctl daemon-reload
systemctl status oracle-database.service --no-pager | head -5
"
```

### 8.4 — Handoff para systemd (padrão obrigatório)

> **PADRÃO OBRIGATÓRIO** (ver CLAUDE.md § Regras Operacionais): o listener e o banco foram iniciados manualmente ao longo desta atividade (Fases 3-6), fora do controle do systemd. Um `systemctl restart` num serviço "inactive" só executa o `ExecStart`, que falha com "already started" (TNS-01106 no listener) porque o processo real já está rodando por fora. Parar manualmente e subir via `systemctl start` para o systemd assumir.

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl stop
'"
$SSH 'cat > /tmp/stop_cdb.sql << '"'"'ENDSQL'"'"'
shutdown immediate
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/stop_cdb.sql"'

${SSH} "
systemctl reset-failed oracle-listener.service oracle-database.service 2>/dev/null
systemctl start oracle-listener.service
sleep 3
systemctl start oracle-database.service
sleep 5
systemctl is-active oracle-listener oracle-database
"
```

Reabrir e persistir a PDB após o restart via systemd (o `dbstart` não abre PDBs sozinho — repetir o passo 6.4):
```bash
${SSH} 'cat > /tmp/openpdb2.sql << '"'"'ENDSQL'"'"'
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
SELECT con_id, name, open_mode FROM v$pdbs ORDER BY con_id;
EXIT;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_19C_PATCHED}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/openpdb2.sql"'
```

### 8.5 — Confirmar .bash_profile e variáveis ao logar como oracle

```bash
${SSH} "su - ${ORACLE_USER} -c 'echo ORACLE_HOME=\$ORACLE_HOME; echo ORACLE_SID=\$ORACLE_SID; sqlplus -v'"
```

Esperado: `ORACLE_HOME=.../dbhome_1`, `ORACLE_SID=CDBORCL`, versão Oracle 19c.

---

## Resultado esperado

- CDB `CDBORCL` com PDB `ORCLPDB` em `OPEN_MODE=READ WRITE`
- Oracle Home único: `dbhome_2` (19.31.0.0.0 com RU p39034528)
- `db_1` e `dbhome_1` removidos, ~14 GB liberados
- OMF ativo (`db_create_file_dest=${ORADATA_DIR}/CDBORCL`) em CDB\$ROOT e ORCLPDB
- Datafiles/tempfile da PDB em `${ORADATA_DIR}/CDBORCL/ORCLPDB/` (não mais em `${ORADATA_DIR}/${ORACLE_SID}/`), diretório antigo removido
- `/etc/oratab`: `CDBORCL:${ORACLE_HOME_19C_PATCHED}:Y`
- `.bash_profile` do oracle: `ORACLE_HOME=dbhome_2`, `ORACLE_SID=CDBORCL`
- Listener em `dbhome_2` com serviços `CDBORCL` e `orclpdb`
- systemd atualizado para `CDBORCL` + `dbhome_2`

## Lições aprendidas (AutoUpgrade 26.x)

- **`target_cdb=NEW` não existe em 26.x** — 26.x tenta conectar num banco chamado "NEW". O CDB alvo deve existir e estar `OPEN`.
- **`global.global_log_dir`** — parâmetro renomeado de `global.autoupg_log_dir` na versão 26.x.
- **OPatch CheckSystemSpace** — com 3 homes simultâneos em /u01 de 20 GB, o OPatch pode falhar por falta de espaço. Remover homes obsoletos antes de aplicar patches.
- **DBCA timeout de SSH** — o DBCA para criar CDB demora 10-15 min. Monitorar via `pmon` e logs em vez de aguardar o retorno SSH.
- **`-clear_recovery_data`** — necessário entre atividades para limpar estado de jobs anteriores que usaram o mesmo SID.
