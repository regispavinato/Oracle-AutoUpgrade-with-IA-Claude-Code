---
description: Atividade 6 — Migrar banco Oracle 19c para Oracle 23ai (23.26.1) usando AutoUpgrade
---

# Atividade 6 — Migrar 19c → Oracle 23ai (23.26.1) com AutoUpgrade

## Contexto
O AutoUpgrade realiza o upgrade completo do banco de 19c para 23ai em modo `deploy`. O banco fonte é o CDB `CDBORCL` (após Atividade 5). O SID real após a Atividade 5 é **CDBORCL** — o `.env` tem `ORACLE_SID=ORCL` que está errado para esta atividade.

## Pré-requisitos
- Oracle 23ai instalado em `${ORACLE_HOME_23AI}` na VM
- Gold Image 23ai extraída e instalada com root.sh executado
- Banco CDBORCL rodando (19c, CDB com ORCLPDB)
- autoupgrade.jar versão compatível com 23ai (23.x) em `${AUTOUPGRADE_JAR}`
- **`${GOLD_IMAGE_23AI}` copiada para `/install` na VM** (não é feito automaticamente por
  nenhuma skill anterior — verificar/copiar antes de começar, ex. via `scp`)
- **Chave SSH configurada para o usuário `oracle`, não só `root`** (2026-08-13: esta é a
  primeira skill do curso que conecta como `oracle@${VM_IP}` diretamente — todas as
  anteriores usam `root@${VM_IP}` + `su - oracle -c '...'`). Se `ssh -i ${SSH_KEY}
  oracle@${VM_IP}` falhar com `Permission denied (publickey...)`, copiar a
  `authorized_keys` do root:
  ```bash
  ${SSH_ROOT} "
  mkdir -p /home/${ORACLE_USER}/.ssh && chmod 700 /home/${ORACLE_USER}/.ssh
  cp /root/.ssh/authorized_keys /home/${ORACLE_USER}/.ssh/authorized_keys
  chown -R ${ORACLE_USER}:${ORACLE_GROUP} /home/${ORACLE_USER}/.ssh
  chmod 600 /home/${ORACLE_USER}/.ssh/authorized_keys
  "
  ```

## Carregar variáveis

Leia `.env` com o Read tool. Nunca exiba senhas.

```bash
SSH="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 oracle@${VM_IP}"
SSH_ROOT="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 root@${VM_IP}"

# SID real após Atividade 5 — ignorar ORACLE_SID do .env (que é ORCL)
REAL_SID=CDBORCL
# AutoUpgrade 26.x requer Java 11 — usar o JDK embutido no Oracle Home 23ai
JAVA="${ORACLE_HOME_23AI}/jdk/bin/java"
```

---

## FASE 0 — Pré-verificações obrigatórias

Execute estas verificações **antes de iniciar o upgrade**. Falhas aqui causam problemas graves no meio do processo.

### 0.1 — Confirmar SID real e Oracle Home atual

```bash
${SSH} "
echo '=== oratab ==='
grep -v '^#' /etc/oratab | grep -v '^$'
echo '=== Processos pmon ==='
ps aux | grep pmon | grep -v grep
"
```

Confirmar: SID é `CDBORCL` rodando. Se `.env` tiver `ORACLE_SID=ORCL`, **ignorar** — usar `CDBORCL` nesta atividade.

### 0.2 — Verificar FRA e expandir preventivamente

O catupgrd gera 10-15 GB de redo em uma VM pequena. A FRA **deve ter ao menos 20 GB livres** antes de iniciar.

```bash
${SSH} "
export ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
-- Ver uso atual da FRA
SELECT name,
       ROUND(space_limit/1073741824,1) AS limit_gb,
       ROUND(space_used/1073741824,1)  AS used_gb,
       ROUND(space_used/space_limit*100,1) AS pct_used
FROM v\$recovery_file_dest;

-- Ver espaço em disco no filesystem
EOF
df -h ${FRA_DIR}
"
```

Se `pct_used > 60%` ou `limit_gb < 20`, expandir antes de prosseguir:

```bash
${SSH} "
export ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
-- Expandir FRA para 20 GB (ajustar se o filesystem /u02 tiver mais espaço)
ALTER SYSTEM SET db_recovery_file_dest_size = 20G SCOPE=BOTH;
SELECT ROUND(space_limit/1073741824,1) AS novo_limite_gb FROM v\$recovery_file_dest;
EOF
"
```

### 0.3 — Verificar e copiar spfile para o Oracle Home 23ai

O AutoUpgrade pode falhar no restart se o spfile não estiver no 23ai home. Copiar preventivamente:

```bash
${SSH} "
SPFILE_SRC=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)/dbs/spfileCDBORCL.ora
SPFILE_DST=${ORACLE_HOME_23AI}/dbs/spfileCDBORCL.ora

if [ -f \"\$SPFILE_SRC\" ] && [ ! -f \"\$SPFILE_DST\" ]; then
  cp \"\$SPFILE_SRC\" \"\$SPFILE_DST\"
  echo 'spfile copiado para 23ai home'
elif [ -f \"\$SPFILE_DST\" ]; then
  echo 'spfile ja existe no 23ai home'
else
  echo 'AVISO: spfile nao encontrado em nenhum home — verificar localizacao'
  find /u01/app/oracle -name 'spfileCDBORCL.ora' 2>/dev/null
fi
"
```

### 0.3b — Confirmar ARCHIVELOG (obrigatório para o AutoUpgrade)

**Nota (2026-08-13):** o AutoUpgrade exige o banco em `ARCHIVELOG` (com FRA configurada,
ver 0.2) para tirar o guarantee restore point — sem isso `-mode analyze` falha no precheck
com `ERROR ... Enable the database in ARCHIVE LOG mode`. A skill `05-converter-multitenant`
já cria o CDB em `ARCHIVELOG`, mas confirmar aqui é barato e evita descobrir isso só depois
do `analyze` falhar:

```bash
${SSH} "
export ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
SELECT log_mode FROM v\$database;
EOF
"
```
Se retornar `NOARCHIVELOG`, habilitar (banco precisa ficar em `MOUNT`, breve indisponibilidade):
```bash
${SSH} "
export ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
ALTER SYSTEM REGISTER;
SELECT log_mode FROM v\$database;
EOF
"
```

### 0.4 — Verificar espaço em /u01

```bash
${SSH} "df -h /u01; echo '--- (precisa de ao menos 500 MB livres para logs)'"
```

### 0.5 — Verificar Java

```bash
${SSH} "java -version 2>&1 | head -2"
```

Se Java não estiver presente: `dnf install -y java-17-openjdk`.

### 0.6 — Limpeza preventiva de jobs anteriores do AutoUpgrade (padrão, sempre executar)

> **PADRÃO OBRIGATÓRIO** (ver CLAUDE.md § Regras Operacionais): antes de QUALQUER `-mode analyze`/`-mode deploy`, descobrir e limpar o job mais recente do SID `CDBORCL` — não esperar o erro "unfinished execution" para só então reagir. Jobs de atividades anteriores (ex. Atividade 5) ficam indexados pelo mesmo SID e bloqueiam a nova execução.

```bash
${SSH} "
LAST_JOB=\$(ls -td /u01/app/oracle/cfgtoollogs/autoupgrade*/CDBORCL/CDBORCL/*/ 2>/dev/null | head -1 | xargs -n1 basename 2>/dev/null)
echo \"Ultimo job encontrado para SID CDBORCL: \${LAST_JOB:-nenhum}\"
if [ -n \"\$LAST_JOB\" ]; then
  su - ${ORACLE_USER} -c \"
  export ORACLE_HOME=\\\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
  export ORACLE_SID=CDBORCL
  export PATH=\\\$ORACLE_HOME/bin:\\\$PATH
  ${JAVA} -jar ${AUTOUPGRADE_JAR} -config ${INSTALL_DIR}/autoupgrade_upgrade23ai.cfg -clear_recovery_data -jobs \$LAST_JOB
  \" 2>&1 | tail -5
fi
"
```

(`-clear_recovery_data` retornando "not found" é inofensivo — o `.cfg` ainda não existe na primeira execução; rodar de novo após criar o `.cfg` na Fase 2 se necessário.)

---

## FASE 1 — Instalar Oracle Home 23ai

### 1.1 — Verificar se o home 23ai já existe

```bash
${SSH} "
if [ -f ${ORACLE_HOME_23AI}/bin/sqlplus ]; then
  echo 'Oracle Home 23ai ja instalado:'
  ${ORACLE_HOME_23AI}/bin/sqlplus -v 2>/dev/null
else
  echo 'Oracle Home 23ai NAO encontrado. Iniciando instalacao...'
fi
"
```

### 1.2 — Criar diretório e extrair Gold Image 23ai (se necessário)

```bash
${SSH_ROOT} "
if [ ! -d ${ORACLE_HOME_23AI}/bin ]; then
  mkdir -p ${ORACLE_HOME_23AI}
  chown -R ${ORACLE_USER}:${ORACLE_GROUP} ${ORACLE_HOME_23AI}
  ls -lh ${INSTALL_DIR}/${GOLD_IMAGE_23AI} 2>/dev/null || \
    echo 'ERRO: Gold Image 23ai nao encontrada em ${INSTALL_DIR}/${GOLD_IMAGE_23AI}'
fi
"
```

```bash
${SSH} "
if [ ! -f ${ORACLE_HOME_23AI}/bin/sqlplus ]; then
  cd ${ORACLE_HOME_23AI}
  unzip -q ${INSTALL_DIR}/${GOLD_IMAGE_23AI}
  echo 'Gold Image extraida'
fi
"
```

### 1.3 — Instalar Oracle 23ai (software only)

```bash
${SSH} "
if [ ! -f ${ORACLE_HOME_23AI}/bin/sqlplus ]; then
  ${ORACLE_HOME_23AI}/runInstaller -silent -ignorePrereqFailure \
    oracle.install.option=INSTALL_DB_SWONLY \
    ORACLE_HOSTNAME=${VM_HOSTNAME} \
    UNIX_GROUP_NAME=${ORACLE_GROUP} \
    INVENTORY_LOCATION=${ORACLE_INVENTORY} \
    ORACLE_HOME=${ORACLE_HOME_23AI} \
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
fi
" 2>&1 | tail -10
```

### 1.4 — Executar root.sh do 23ai

```bash
${SSH_ROOT} "[ -f ${ORACLE_HOME_23AI}/root.sh ] && ${ORACLE_HOME_23AI}/root.sh || echo 'root.sh ja executado'"
```

### 1.5 — Verificar versão do Oracle 23ai instalado

```bash
${SSH} "${ORACLE_HOME_23AI}/bin/sqlplus -v 2>/dev/null"
```

---

## FASE 2 — Preparar upgrade com AutoUpgrade

### 2.1 — Confirmar autoupgrade.jar (NÃO sobrescrever com o do Oracle Home 23ai)

**Nota (2026-08-13):** a versão anterior deste passo copiava
`${ORACLE_HOME_23AI}/rdbms/admin/autoupgrade.jar` por cima do `${AUTOUPGRADE_JAR}` atual —
isso é uma **regressão**, não uma atualização: o jar embutido no Gold Image 23ai é o que
veio empacotado na release (`26.5.260117`, o próprio jar avisou "older than 180 days"),
mais antigo que o jar já baixado via `wget` na Atividade 2 (`26.5.260807`). Confirmado nesta
sessão: o jar do Oracle Home também tem `build.supported_target_versions` sem o `26`
(`12.2,18,19,21,23` vs `12.2,18,19,21,23,26` do baixado). Não sobrescrever — só confirmar
que o jar atual já é recente o bastante; se `-version` acusar mais de 180 dias, baixar de
novo (mesmo comando da Atividade 2, Fase 2.2):

```bash
${SSH} "java -jar ${AUTOUPGRADE_JAR} -version 2>/dev/null | head -3"
```
Se a versão for antiga ou o jar não existir:
```bash
${SSH_ROOT} "
wget -O ${AUTOUPGRADE_JAR} https://download.oracle.com/otn-pub/otn_software/autoupgrade.jar 2>&1 | tail -5
chown ${ORACLE_USER}:${ORACLE_GROUP} ${AUTOUPGRADE_JAR}
java -jar ${AUTOUPGRADE_JAR} -version 2>/dev/null | head -3
"
```

### 2.2 — Determinar Oracle Home fonte atual

```bash
${SSH} "
ORACLE_HOME_SOURCE=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
echo \"Home fonte (de oratab): \$ORACLE_HOME_SOURCE\"
echo \"SID: CDBORCL\"
"
```

### 2.3 — Criar config de upgrade para 23ai

Exibir para o usuário verificar antes de criar:

```
global.global_log_dir=/u01/app/oracle/cfgtoollogs/autoupgrade_upgrade23ai

upg1.source_home=<HOME_FONTE_DO_ORATAB>
upg1.target_home=${ORACLE_HOME_23AI}
upg1.sid=CDBORCL
upg1.run_utlrp=yes
upg1.timezone_upg=yes
upg1.target_version=23.26.1
```

```bash
${SSH} "
ORACLE_HOME_SOURCE=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)

cat > ${INSTALL_DIR}/autoupgrade_upgrade23ai.cfg << EOF
global.global_log_dir=/u01/app/oracle/cfgtoollogs/autoupgrade_upgrade23ai

upg1.source_home=\${ORACLE_HOME_SOURCE}
upg1.target_home=${ORACLE_HOME_23AI}
upg1.sid=CDBORCL
upg1.run_utlrp=yes
upg1.timezone_upg=yes
upg1.target_version=${ORACLE_23AI_VERSION}
EOF
chown ${ORACLE_USER}:${ORACLE_GROUP} ${INSTALL_DIR}/autoupgrade_upgrade23ai.cfg
cat ${INSTALL_DIR}/autoupgrade_upgrade23ai.cfg
"
```

---

## FASE 3 — Analisar e fazer fixups

### 3.1 — Modo analyze

```bash
${SSH} "
export ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

${JAVA} -jar ${AUTOUPGRADE_JAR} \
  -config ${INSTALL_DIR}/autoupgrade_upgrade23ai.cfg \
  -mode analyze \
  -noconsole
" 2>&1 | tail -30
```

### 3.2 — Revisar problemas encontrados

```bash
${SSH} "
find /u01/app/oracle/cfgtoollogs/autoupgrade_upgrade23ai -name '*.log' | \
  xargs ls -t 2>/dev/null | head -1 | \
  xargs grep -E 'FAIL|ERROR|WARNING|CHECK_FAILED' 2>/dev/null | head -30
"
```

---

## FASE 4 — Executar upgrade para 23ai

### 4.1 — Executar upgrade (deploy) em background

O deploy leva 60-90 minutos. Rodar em background com nohup para evitar perda por queda de SSH.

```bash
${SSH} "
export ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

mkdir -p ${INSTALL_DIR}/logs
nohup ${JAVA} -jar ${AUTOUPGRADE_JAR} \
  -config ${INSTALL_DIR}/autoupgrade_upgrade23ai.cfg \
  -mode deploy \
  -noconsole > ${INSTALL_DIR}/logs/autoupgrade_deploy.log 2>&1 &
echo \"AutoUpgrade PID=\$!\"
echo \"Log: ${INSTALL_DIR}/logs/autoupgrade_deploy.log\"
"
```

### 4.2 — Monitorar progresso

```bash
${SSH} "
tail -20 ${INSTALL_DIR}/logs/autoupgrade_deploy.log 2>/dev/null
echo '--- job status ---'
find /u01/app/oracle/cfgtoollogs/autoupgrade_upgrade23ai -name '*.log' -newer /tmp -exec tail -5 {} \; 2>/dev/null | head -30
"
```

### 4.3 — Monitorar FRA durante o upgrade

**Executar a cada 10-15 minutos enquanto o upgrade roda:**

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
SELECT ROUND(space_used/space_limit*100,1) AS fra_pct,
       ROUND(space_used/1073741824,1) AS used_gb,
       ROUND(space_limit/1073741824,1) AS limit_gb
FROM v\$recovery_file_dest;
EOF
"
```

Se `fra_pct > 85%`, expandir imediatamente (ver Fase 0.2).

---

## FASE 4B — PLANO B: Se AutoUpgrade Java travar (deadlock)

O AutoUpgrade pode travar após o upgrade do CDB$ROOT com um deadlock de pipe stdin (bug conhecido).
Sintoma: logs param de crescer, Java consome 0% CPU mas não avança para PDB$SEED.

**Diagnóstico:**
```bash
${SSH} "
ps aux | grep -E 'java|sqlplus' | grep -v grep
# Se sqlplus estiver em wait > 10 min após 'Dispatcher finished':
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
SELECT sid, event, seconds_in_wait FROM v\$session
WHERE wait_class NOT IN ('Idle','Other')
ORDER BY seconds_in_wait DESC FETCH FIRST 5 ROWS ONLY;
EOF
"
```

**Ação — upgrade manual com catctl.pl:**

1. Matar o processo Java:
```bash
${SSH} "kill \$(ps aux | grep autoupgrade | grep java | awk '{print \$2}'); echo 'Java encerrado'"
```

2. Verificar estado dos containers:
```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
SELECT name, open_mode FROM v\$pdbs;
SELECT version_full FROM v\$instance;
EOF
"
```

3. Reiniciar em modo UPGRADE e abrir todos os PDBs:
```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
SHUTDOWN IMMEDIATE;
STARTUP UPGRADE;
ALTER PLUGGABLE DATABASE ALL OPEN UPGRADE;
SELECT name, open_mode FROM v\$pdbs;
EOF
"
```

4. Rodar catupgrd com catctl.pl (NÃO usar catcon.pl nem rodar catupgrd.sql direto).
**Sempre `< /dev/null` no nohup** (2026-08-13: sem isso, o `catctl.pl`/seus `sqlplus`
filhos podem travar esperando um stdin que nunca chega — foi confirmado nesta sessão que
isso, e não paralelismo, era a causa de dois travamentos reais em fases diferentes; com
`< /dev/null` o processo completo rodou sem travar nenhuma vez):

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$ORACLE_HOME/perl/bin:\$PATH
mkdir -p /tmp/upg_manual_logs

nohup \$ORACLE_HOME/perl/bin/perl \$ORACLE_HOME/rdbms/admin/catctl.pl \
  -n 2 \
  -l /tmp/upg_manual_logs \
  \$ORACLE_HOME/rdbms/admin/catupgrd.sql < /dev/null > /tmp/upg_manual_logs/catctl_all.log 2>&1 &
disown
echo \"catctl.pl PID=\$!\"
echo \"Log: /tmp/upg_manual_logs/catctl_all.log\"
"
```

5. Monitorar progresso (verificar a cada 10 min):
```bash
${SSH} "
grep -E '^(Serial|Parallel|Restart).*Phase.*Time:' /tmp/upg_manual_logs/catctl_all.log 2>/dev/null | tail -10
echo '---'
tail -3 /tmp/upg_manual_logs/catctl_all.log
"
```

**Nota crítica (2026-08-13) — NÃO confiar só no log parado para decidir que travou.** Fases
`Files:1` (um arquivo só) que rodam algo pesado — ex. `dbms_utility.validate` visto nesta
sessão — não imprimem nada até terminar, e legitimamente levaram até ~7 minutos numa VM
pequena (6 vCPU/6GB compartilhados). Confundir isso com o deadlock real (item 4 acima)
e matar o processo interrompe trabalho válido e obriga recomeçar o container inteiro. Antes
de matar, confirmar que é deadlock de verdade checando `v$session`:
```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
SELECT sid, status, last_call_et, sql_id FROM v\$session WHERE username='SYS' ORDER BY last_call_et DESC;
EOF
"
```
Rodar duas vezes com alguns minutos de intervalo: se alguma sessão está `ACTIVE` com
`last_call_et` **crescendo** entre as duas checagens, está trabalhando de verdade — deixar
rodar. Só considerar travado (deadlock real) se as sessões relevantes ficarem `INACTIVE`
sem nenhum avanço no log por muito tempo (bem mais que os ~7 min de uma validação pesada).

6. Após conclusão (procurar "Grand Total Upgrade Time" no log — não há uma linha `RC=`
   separada nesta versão, o sucesso é a presença dessa linha final sem "ORA-" antes dela):
```bash
${SSH} "grep -E 'Grand Total Upgrade Time|ORA-' /tmp/upg_manual_logs/catctl_all.log"
```

7. Abrir banco normalmente e continuar com FASE 5.

---

## FASE 5 — Verificação e pós-upgrade

### 5.1 — Verificar versão e estado dos containers

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
SELECT version_full FROM v\$instance;
SELECT name, open_mode, restricted FROM v\$pdbs;
SELECT cid, version_full, status FROM sys.registry\$ ORDER BY cid;
EOF
"
```

Todos os componentes devem ter `status=1` (VALID) e `version_full=23.26.1.0.0`.

### 5.2 — utlrp com Resource Manager desabilitado

**IMPORTANTE:** O Resource Manager pode matar os processos paralelos do utlrp (ORA-12751).
Desabilitar antes de rodar:

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

# Abrir ORCLPDB e persistir estado (PADRÃO OBRIGATÓRIO — sempre as duas juntas, ver CLAUDE.md)
sqlplus -s / as sysdba << 'EOF'
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
-- Desabilitar Resource Manager
ALTER SYSTEM SET resource_manager_plan='' SCOPE=BOTH;
EOF

# Rodar utlrp em background
nohup sqlplus -s / as sysdba > /tmp/utlrp_postupgrade.log 2>&1 << 'EOF' &
ALTER SESSION SET CONTAINER=CDB\$ROOT;
@?/rdbms/admin/utlrp.sql
ALTER SESSION SET CONTAINER=ORCLPDB;
@?/rdbms/admin/utlrp.sql
EOF
echo \"utlrp PID=\$!\"
"
```

Verificar resultado após ~5 min:
```bash
${SSH} "
grep -A2 'OBJECTS WITH ERRORS\|ERRORS DURING' /tmp/utlrp_postupgrade.log 2>/dev/null
tail -5 /tmp/utlrp_postupgrade.log
"
```

Ambos devem ser `0`.

### 5.3 — datapatch

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
\$ORACLE_HOME/OPatch/datapatch -verbose 2>&1 | tail -20
"
```

### 5.4 — Reconfirmar OPEN + SAVE STATE da PDB (após datapatch)

O `datapatch`/`utlrp` pode ter reiniciado containers — reconfirmar o padrão (OPEN + SAVE STATE sempre juntos, idempotente):

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
SELECT name, open_mode FROM v\$pdbs;
EOF
"
```

### 5.5 — Atualizar /etc/oratab e serviços systemd

**Nota (2026-08-13):** o `sed 's|ORACLE_HOME=.*|...|g'` abaixo só substitui linhas que
contêm literalmente o texto `ORACLE_HOME=` (a linha `Environment=ORACLE_HOME=...`) — ele
**não** toca `ExecStart=`/`ExecStop=`/`Environment=PATH=`, que têm o caminho do Oracle Home
**hardcoded diretamente**, sem o prefixo `ORACLE_HOME=`. Resultado real observado: o
`Environment=ORACLE_HOME=` ficava correto (23ai) mas o `ExecStart=.../bin/dbstart ...`
continuava chamando o `dbstart` do home 19c antigo — o serviço reportava `active` sem
nenhum `pmon` de pé. Buscar pelo **caminho antigo completo** (que aparece em todas as
linhas relevantes), não pelo nome da variável:

```bash
${SSH_ROOT} "
# Atualizar oratab para apontar para 23ai
sed -i 's|CDBORCL:.*:Y|CDBORCL:${ORACLE_HOME_23AI}:Y|' /etc/oratab
grep CDBORCL /etc/oratab

# Atualizar serviços systemd — substituir o CAMINHO ANTIGO completo, não so a var ORACLE_HOME=
for svc in oracle-database.service oracle-listener.service; do
  [ -f /etc/systemd/system/\$svc ] && \
    sed -i 's|${ORACLE_HOME_19C_PATCHED}|${ORACLE_HOME_23AI}|g; s|ORACLE_SID=.*|ORACLE_SID=CDBORCL|g' \
    /etc/systemd/system/\$svc
done
systemctl daemon-reload
grep -E 'ORACLE_HOME|ExecStart|ExecStop' /etc/systemd/system/oracle-database.service
echo 'systemd atualizado'
"
```

### 5.5b — Migrar o listener para o home 23ai (faltava nesta skill, mesmo gap da Atividade 4)

**Nota (2026-08-13):** o `catctl.pl`/AutoUpgrade não migram o listener — ele continua
apontando para `${ORACLE_HOME_19C_PATCHED}` mesmo depois do `oracle-listener.service` ter
o `ORACLE_HOME` atualizado na 5.5 (o service só muda o que o systemd usa no próximo start;
o listener já rodando continua no home antigo até ser reiniciado explicitamente no home
novo). Sem este passo, `lsnrctl status` mostra `Parameter File` no home 19c e a versão do
TNSLSNR fica desatualizada:

```bash
${SSH} "
export ORACLE_HOME=<home_atual_do_listener_ex_19c_patched>
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl stop

mkdir -p ${ORACLE_HOME_23AI}/network/admin
cp <home_atual>/network/admin/listener.ora ${ORACLE_HOME_23AI}/network/admin/listener.ora
cp <home_atual>/network/admin/tnsnames.ora ${ORACLE_HOME_23AI}/network/admin/tnsnames.ora 2>/dev/null || true

export ORACLE_HOME=${ORACLE_HOME_23AI}
export PATH=\$ORACLE_HOME/bin:\$PATH
lsnrctl start
"
```
Registrar os serviços no listener novo:
```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
ALTER SYSTEM REGISTER;
EOF
lsnrctl status 2>&1 | grep -E 'Parameter File|Version|Service'
"
```
Verificar: `Parameter File` aponta pro `${ORACLE_HOME_23AI}`, `Version TNSLSNR` mostra
`23.26.1.0.0`, serviços `CDBORCL`/`CDBORCLXDB`/`orclpdb` presentes.

### 5.6 — Atualizar .bash_profile do oracle

```bash
${SSH} "
sed -i 's|export ORACLE_HOME=.*|export ORACLE_HOME=${ORACLE_HOME_23AI}|' ~/.bash_profile
sed -i 's|export ORACLE_SID=.*|export ORACLE_SID=CDBORCL|' ~/.bash_profile
grep -E 'ORACLE_HOME|ORACLE_SID|PATH' ~/.bash_profile
"
```

### 5.7 — Remover parâmetros depreciados do spfile (evita duplo-start do dbstart)

**Nota (2026-08-13):** o spfile copiado do 19c pra cá (Fase 0.3) carrega parâmetros que o
23ai considera depreciados (`audit_file_dest`, `audit_trail` — podem variar). Todo boot
gera `ORA-32004: obsolete or deprecated parameter(s)` como aviso — inofensivo por si só,
mas foi observado que isso faz o `dbstart` do systemd **iniciar a instância duas vezes**
em sequência (a segunda tentativa falha com `ORA-01081: cannot start already-running
ORACLE`, deixando uma SGA órfã sem `pmon` vivo mesmo com `systemctl is-active` dizendo
`active`). Resetar os parâmetros do spfile resolve — confirmado nesta sessão que o
`systemctl restart` ficou limpo (um único `ORACLE instance started`) depois disso:

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
-- Ver quais parametros estao marcados como depreciados (olhar o alert log tambem):
-- grep -A5 'Deprecated system parameters' \$ORACLE_BASE/diag/rdbms/cdborcl/CDBORCL/trace/alert_CDBORCL.log
ALTER SYSTEM RESET audit_file_dest SCOPE=SPFILE SID='*';
ALTER SYSTEM RESET audit_trail SCOPE=SPFILE SID='*';
EOF
"
```
Efeito só aparece no próximo restart — se ainda não reiniciou depois da 5.5b, o restart da
5.8 abaixo já pega o spfile limpo.

### 5.8 — Handoff para systemd (padrão obrigatório)

> Mesma regra das atividades 1/4/5 (ver `CLAUDE.md` § Regras Operacionais): usar sempre
> `systemctl restart`, nunca `start` — e confirmar com `ps aux | grep pmon`, não só
> `systemctl is-active`, que não garante processo real de pé.

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << '"'"'EOF'"'"'
SHUTDOWN IMMEDIATE;
EOF
'"
${SSH_ROOT} "
systemctl reset-failed oracle-listener.service oracle-database.service 2>/dev/null
systemctl restart oracle-listener.service
sleep 3
systemctl restart oracle-database.service
sleep 10
systemctl is-active oracle-listener oracle-database
ps aux | grep pmon | grep -v grep
"
```
Reabrir e persistir a PDB depois do restart (o `dbstart` não abre PDBs sozinho):
```bash
${SSH} 'cat > /tmp/openpdb_final.sql << '"'"'ENDSQL'"'"'
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
ALTER SYSTEM REGISTER;
SELECT name, open_mode FROM v\$pdbs;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME=${ORACLE_HOME_23AI}; export ORACLE_SID=CDBORCL; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/openpdb_final.sql"'
```

---

## Recuperação de emergência

### Instância não aceita conexões (ROW CACHE LOCK / PROCESSES esgotado)

Sintoma: `sqlplus` trava ao tentar conectar, timeout após 10-15s.

```bash
# Conectar em modo prelim (ignora locks do dicionário)
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -prelim / as sysdba << 'EOF'
SHUTDOWN ABORT;
EOF
"

# Iniciar normalmente
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
STARTUP;
SELECT status FROM v\$instance;
EOF
"
```

### FRA cheia durante upgrade (archiver travado)

Sintoma: `wait_class='Configuration'`, evento `log file switch (archiving needed)`, `seconds_in_wait > 300`.

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
sqlplus -s / as sysdba << 'EOF'
-- Expandir FRA em 10 GB a mais (verificar espaço em disco antes)
ALTER SYSTEM SET db_recovery_file_dest_size =
  (SELECT space_limit/1073741824 + 10 || 'G' FROM v\$recovery_file_dest) SCOPE=BOTH;
SELECT ROUND(space_limit/1073741824,1) AS novo_gb FROM v\$recovery_file_dest;
EOF
"
```

---

## Resultado esperado

- Banco CDBORCL versão `23.26.1.0.0` em `OPEN_MODE=READ WRITE`
- PDB ORCLPDB `READ WRITE`, não restricted
- Todos os componentes `VALID` (status=1 em `sys.registry$`)
- `datapatch` executado com sucesso
- `/etc/oratab` e serviços systemd apontando para `${ORACLE_HOME_23AI}`
- ORCLPDB configurado para abrir automaticamente no startup
