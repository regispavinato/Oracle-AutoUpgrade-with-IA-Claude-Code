---
description: Atividade 1 — Preparação completa do Oracle Linux 8.10, LVM /u02, instalação Oracle 19c non-CDB, banco ORCL, listener, startup e bash_profile
---

# Atividade 1 — Preparação do Ambiente e Instalação Oracle 19c non-CDB

## Pré-requisitos
- `.env` preenchido e SSH configurado (`/oracle-imersao setup-ssh`)
- Gold Image 19c presente em `${INSTALL_DIR}/${GOLD_IMAGE_19C}` na VM
- VM acessível via SSH como root

## Carregar variáveis

Leia `.env` com o Read tool e extraia todas as variáveis. Nunca exiba senhas no output.

Defina internamente:
```
SSH="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR root@${VM_IP}"
```

---

## FASE 1 — Sistema Operacional

### 1.1 — Atualizar /etc/hosts

```bash
${SSH} "grep -q '${VM_HOSTNAME}' /etc/hosts || echo '${VM_IP}  ${VM_HOSTNAME}  imersao' >> /etc/hosts && cat /etc/hosts"
```
Verificar: hostname deve aparecer no arquivo.

### 1.2 — Instalar oracle-database-preinstall-19c

```bash
${SSH} "dnf install -y oracle-database-preinstall-19c 2>&1 | tail -5"
```
Este pacote cria automaticamente o usuário `oracle`, grupos `oinstall/dba`, ajusta kernel parameters e limits.

### 1.3 — Desativar SELinux

```bash
${SSH} "
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
setenforce 0 2>/dev/null || true
getenforce
"
```
Verificar: retorno deve ser `Permissive` ou `Disabled`.

### 1.4 — Desativar firewalld

```bash
${SSH} "
systemctl disable --now firewalld
systemctl is-active firewalld || echo 'firewalld desativado com sucesso'
"
```

### 1.5 — Atualizar todos os pacotes (yum update)

```bash
${SSH} "dnf update -y 2>&1 | tail -10"
```
Pode demorar vários minutos. Se houver atualização de kernel, executar reboot e reconectar:

```bash
${SSH} "needs-restarting -r 2>/dev/null; echo EXIT_CODE=$?"
```
**Nota (2026-08-13):** em instalação `@^minimal-environment` (skill `00-criar-vm`), o
comando `needs-restarting` não existe (pacote `dnf-utils`/`yum-utils` não vem no minimal) —
retorna `EXIT_CODE=127` (command not found), não `0`/`1`. Nesse caso, tratar `127` como
"reboot necessário": se o `dnf update` da 1.5 listou `kernel*` em "Installed:", reiniciar
direto sem depender desse check.

Se a saída indicar reboot necessário (EXIT_CODE=1, ou 127 pelo motivo acima):
```bash
${SSH} "reboot" 2>/dev/null || true
```
Aguardar 60 segundos e testar reconexão:
```bash
sleep 60
${SSH} "echo 'Reconectado apos reboot'"
```
Tentar até 5 vezes com intervalo de 20s se não conectar de imediato.

### 1.6 — Ajustar permissão do /install

```bash
${SSH} "chown -R oracle:oinstall ${INSTALL_DIR} && ls -ld ${INSTALL_DIR}"
```

---

## FASE 2 — LVM para /u02

### 2.1 — Identificar o disco de dados (não usado pelo SO)

```bash
${SSH} "lsblk -d -o NAME,SIZE,TYPE,MOUNTPOINT | grep disk"
```

**IMPORTANTE:** Identificar automaticamente qual disco NÃO está em uso pelo SO. Executar:

```bash
${SSH} "lsblk -d -o NAME,SIZE,TYPE | grep disk | while read name size type; do
  usado=$(lsblk -no MOUNTPOINT /dev/\$name 2>/dev/null | grep -v '^$' | head -1)
  if [ -z \"\$usado\" ] && ! pvs /dev/\$name &>/dev/null; then
    echo \"LIVRE: /dev/\$name (\$size)\"
  else
    echo \"EM USO: /dev/\$name (\$size) <- SO ou LVM\"
  fi
done"
```

Comparar com `DISK_DEVICE` do `.env`. Se divergir, **atualizar o `.env`** antes de prosseguir — nunca usar o disco do SO.

### 2.2 — Criar Physical Volume

```bash
${SSH} "pvcreate ${DISK_DEVICE} && pvs"
```

### 2.3 — Criar Volume Group

```bash
${SSH} "vgcreate ${VG_NAME} ${DISK_DEVICE} && vgs"
```

### 2.4 — Criar Logical Volume (100% do VG)

```bash
${SSH} "lvcreate -l 100%FREE -n ${LV_NAME} ${VG_NAME} && lvs"
```

### 2.5 — Formatar com XFS

```bash
${SSH} "mkfs.xfs /dev/${VG_NAME}/${LV_NAME}"
```

### 2.6 — Criar ponto de montagem e montar

```bash
${SSH} "
mkdir -p ${MOUNT_POINT}
mount /dev/${VG_NAME}/${LV_NAME} ${MOUNT_POINT}
df -h ${MOUNT_POINT}
"
```

### 2.7 — Persistir no /etc/fstab

```bash
${SSH} "
FSTAB_ENTRY='/dev/${VG_NAME}/${LV_NAME}  ${MOUNT_POINT}  xfs  defaults  0 0'
grep -q '${MOUNT_POINT}' /etc/fstab || echo \"\${FSTAB_ENTRY}\" >> /etc/fstab
tail -3 /etc/fstab
"
```
Verificar a montagem persistida:
```bash
${SSH} "mount -a && df -h ${MOUNT_POINT}"
```

### 2.8 — Criar estrutura de diretórios e permissões

```bash
${SSH} "
mkdir -p ${ORADATA_DIR} ${FRA_DIR}
chown -R ${ORACLE_USER}:${ORACLE_GROUP} ${MOUNT_POINT}
chmod -R 775 ${MOUNT_POINT}
ls -la ${MOUNT_POINT}
"
```

---

## FASE 3 — Instalação Oracle 19c

### 3.1 — Criar estrutura de diretórios Oracle

```bash
${SSH} "
mkdir -p ${ORACLE_HOME_19C}
mkdir -p ${ORACLE_INVENTORY}
chown -R ${ORACLE_USER}:${ORACLE_GROUP} /u01/app
chmod -R 775 /u01/app
ls -la /u01/app/oracle/product/19.3.0/
"
```

### 3.2 — Verificar Gold Image

```bash
${SSH} "ls -lh ${INSTALL_DIR}/${GOLD_IMAGE_19C}"
```
Se o arquivo não existir, pausar e instruir o usuário a copiar a Gold Image para `/install` na VM.

### 3.3 — Extrair Gold Image no Oracle Home

```bash
${SSH} "su - ${ORACLE_USER} -c 'cd ${ORACLE_HOME_19C} && unzip -q ${INSTALL_DIR}/${GOLD_IMAGE_19C}' 2>&1 | tail -5"
```
Pode demorar alguns minutos.

### 3.4 — Executar instalação silent (software only)

**NOTA:** O instalador Oracle 19.3 em OL 8.x pode crashar com `NullPointerException` no `supportedOSCheck`. Workaround obrigatório: exportar `CV_ASSUME_DISTID=OEL8.0` antes de executar.

```bash
${SSH} "su - ${ORACLE_USER} -c '
export CV_ASSUME_DISTID=OEL8.0
${ORACLE_HOME_19C}/runInstaller -silent -ignorePrereqFailure \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=${VM_HOSTNAME} \
  UNIX_GROUP_NAME=${ORACLE_GROUP} \
  INVENTORY_LOCATION=${ORACLE_INVENTORY} \
  ORACLE_HOME=${ORACLE_HOME_19C} \
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
' 2>&1 | tail -20"
```
Aguardar conclusão (~5 min). O instalador termina com "Successfully Setup Software."

### 3.5 — Executar scripts de root

```bash
${SSH} "
${ORACLE_INVENTORY}/orainstRoot.sh
${ORACLE_HOME_19C}/root.sh
"
```

---

## FASE 4 — Listener

### 4.1 — Criar listener.ora

```bash
${SSH} "
mkdir -p ${ORACLE_HOME_19C}/network/admin
cat > ${ORACLE_HOME_19C}/network/admin/listener.ora << 'EOF'
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = ${VM_HOSTNAME})(PORT = 1521))
    )
  )

ADR_BASE_LISTENER = ${ORACLE_BASE}
EOF
chown ${ORACLE_USER}:${ORACLE_GROUP} ${ORACLE_HOME_19C}/network/admin/listener.ora
"
```

### 4.2 — Criar tnsnames.ora

```bash
${SSH} "
cat > ${ORACLE_HOME_19C}/network/admin/tnsnames.ora << 'EOF'
${ORACLE_SID} =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = ${VM_HOSTNAME})(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ${ORACLE_SID})
    )
  )
EOF
chown ${ORACLE_USER}:${ORACLE_GROUP} ${ORACLE_HOME_19C}/network/admin/tnsnames.ora
"
```

### 4.3 — Iniciar listener

```bash
${SSH} "su - ${ORACLE_USER} -c 'export ORACLE_HOME=${ORACLE_HOME_19C}; export PATH=\$ORACLE_HOME/bin:\$PATH; lsnrctl start && lsnrctl status'"
```

---

## FASE 5 — Criação do banco ORCL (DBCA silent)

### 5.1 — Executar DBCA

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_BASE=${ORACLE_BASE}
export ORACLE_HOME=${ORACLE_HOME_19C}
export ORACLE_SID=${ORACLE_SID}
export PATH=\$ORACLE_HOME/bin:\$PATH

dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname ${ORACLE_SID} -sid ${ORACLE_SID} -responseFile NO_VALUE \
  -characterSet ${DB_CHARSET} \
  -sysPassword \"${ORACLE_PWD}\" \
  -systemPassword \"${ORACLE_PWD}\" \
  -createAsContainerDatabase false \
  -databaseType MULTIPURPOSE \
  -memoryMgmtType auto_sga \
  -totalMemory ${DB_MEMORY_MB} \
  -storageType FS \
  -datafileDestination \"${ORADATA_DIR}/\" \
  -recoveryAreaDestination \"${FRA_DIR}/\" \
  -enableArchive TRUE \
  -redoLogFileSize ${DB_REDO_SIZE_MB} \
  -emConfiguration NONE \
  -sampleSchema FALSE \
  -ignorePreReqs
' 2>&1 | grep -v '^$'"
```
Pode levar 15-25 minutos. Verificar: "Database creation complete."

### 5.2 — Verificar banco criado

**NOTA:** Nunca usar heredoc inline para queries com `v$database` via SSH — o `$` é interpretado pelo shell e causa `ORA-00942`. Usar arquivo SQL temporário:

```bash
${SSH} 'cat > /tmp/check_db.sql << '"'"'ENDSQL'"'"'
select name, open_mode, cdb from v$database;
exit;
ENDSQL
su - '"${ORACLE_USER}"' -c "export ORACLE_HOME='"${ORACLE_HOME_19C}"'; export ORACLE_SID='"${ORACLE_SID}"'; export PATH=\$ORACLE_HOME/bin:\$PATH; sqlplus -s / as sysdba @/tmp/check_db.sql"'
```
Esperado: `CDB=NO`, `OPEN_MODE=READ WRITE`.

---

## FASE 6 — .bash_profile do oracle

> **Nota (2026-07-19):** não incluir `ORAENV_ASK=NO` / `. oraenv` neste bloco. As skills seguintes (04-07) atualizam `ORACLE_HOME`/`ORACLE_SID` diretamente via `sed` in-place — rodar `oraenv` depois é redundante e, se alguma skill mover essas linhas de posição no arquivo, o `oraenv` roda antes delas serem definidas e imprime um prompt de auto-detecção confuso ("ORACLE_HOME = [/home/oracle] ?") a cada login.

```bash
${SSH} "
cat >> /home/${ORACLE_USER}/.bash_profile << 'EOF'

# Oracle Environment
export ORACLE_BASE=${ORACLE_BASE}
export ORACLE_HOME=${ORACLE_HOME_19C}
export ORACLE_SID=${ORACLE_SID}
export PATH=\$ORACLE_HOME/bin:\$PATH
export LD_LIBRARY_PATH=\$ORACLE_HOME/lib:/lib:/usr/lib
export NLS_DATE_FORMAT='DD-MON-YYYY HH24:MI:SS'
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
EOF
echo 'bash_profile atualizado'
"
```

---

## FASE 7 — Scripts de inicialização automática

### 7.1 — Atualizar /etc/oratab

**Nota (2026-08-13):** o DBCA (Fase 5) já cria a entrada do `${ORACLE_SID}` em `/etc/oratab`
sozinho — mas com a flag `N`, não `Y`. Como o `grep -q ... || echo ... >> oratab` abaixo só
insere se a linha **não existir**, ele não faz nada (a linha já existe, só que errada) e a
flag fica `N`. Isso é silencioso e só aparece depois: `dbstart` (chamado pelo
`oracle-database.service` na Fase 7.3) lê `/etc/oratab` e **pula** qualquer SID marcado `N`
— o serviço systemd "starta" com sucesso sem de fato subir o banco. Sempre forçar a flag
para `Y` explicitamente, não só inserir se ausente:

```bash
${SSH} "
grep -q '^${ORACLE_SID}:' /etc/oratab || echo '${ORACLE_SID}:${ORACLE_HOME_19C}:Y' >> /etc/oratab
sed -i 's|^${ORACLE_SID}:${ORACLE_HOME_19C}:N|${ORACLE_SID}:${ORACLE_HOME_19C}:Y|' /etc/oratab
cat /etc/oratab | grep -v '^#' | grep -v '^$'
"
```

### 7.2 — Serviço systemd para o Listener

```bash
${SSH} "
cat > /etc/systemd/system/oracle-listener.service << 'EOF'
[Unit]
Description=Oracle TNS Listener
After=network.target

[Service]
Type=forking
User=${ORACLE_USER}
Environment=ORACLE_HOME=${ORACLE_HOME_19C}
Environment=PATH=${ORACLE_HOME_19C}/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=${ORACLE_HOME_19C}/bin/lsnrctl start
ExecStop=${ORACLE_HOME_19C}/bin/lsnrctl stop
RemainAfterExit=yes
TimeoutStartSec=90
TimeoutStopSec=60

[Install]
WantedBy=multi-user.target
EOF
"
```

### 7.3 — Serviço systemd para o banco

```bash
${SSH} "
cat > /etc/systemd/system/oracle-database.service << 'EOF'
[Unit]
Description=Oracle Database ${ORACLE_SID}
After=oracle-listener.service
Requires=oracle-listener.service

[Service]
Type=forking
User=${ORACLE_USER}
Environment=ORACLE_HOME=${ORACLE_HOME_19C}
Environment=ORACLE_SID=${ORACLE_SID}
Environment=PATH=${ORACLE_HOME_19C}/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=${ORACLE_HOME_19C}/bin/dbstart ${ORACLE_HOME_19C}
ExecStop=${ORACLE_HOME_19C}/bin/dbshut ${ORACLE_HOME_19C}
RemainAfterExit=yes
TimeoutStartSec=600
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
EOF
"
```

### 7.4 — Habilitar e verificar serviços

```bash
${SSH} "
systemctl daemon-reload
systemctl enable oracle-listener oracle-database
systemctl is-enabled oracle-listener
systemctl is-enabled oracle-database
"
```

### 7.5 — Handoff: parar manual, subir via systemd

Regra do projeto: o listener (Fase 4.3, `lsnrctl start` manual) e o banco (Fase 5.1, subido
pelo próprio DBCA) neste ponto ainda estão rodando **fora** do systemd — mesmo com os
serviços "enabled", o systemd não rastreia processos que ele não iniciou. Parar e religar
via `systemctl` agora, no fim da atividade, evita o `TNS-01106`/"already started" de um
`systemctl restart` futuro achando o serviço "inactive" com a porta já em uso:

```bash
${SSH} "su - ${ORACLE_USER} -c '
export ORACLE_HOME=${ORACLE_HOME_19C}
export ORACLE_SID=${ORACLE_SID}
export PATH=\$ORACLE_HOME/bin:\$PATH
echo shutdown immediate | sqlplus -s / as sysdba
lsnrctl stop
'"
${SSH} "systemctl start oracle-listener oracle-database"
${SSH} "systemctl is-active oracle-listener oracle-database"
${SSH} "ps aux | grep -E 'pmon_${ORACLE_SID}|tnslsnr' | grep -v grep"
```
Verificar: `active`/`active`, e os processos `ora_pmon_${ORACLE_SID}` e `tnslsnr` presentes.

**Pré-requisito para este passo funcionar: a flag do `/etc/oratab` precisa estar `Y`** (ver
nota da Fase 7.1) — `dbstart` (chamado pelo `ExecStart` do `oracle-database.service`) lê
essa flag e **pula silenciosamente** qualquer SID marcado `N`. Sem o fix da 7.1, este passo
mostraria `systemctl is-active` como `active` (o serviço "rodou" o `dbstart` com sucesso)
mas `ps aux` não teria nenhum `ora_pmon_${ORACLE_SID}` — confirmado nesta sessão.

---

## FASE 8 — OpenJDK e bootstrap do autoupgrade.jar

O AutoUpgrade requer Java 11 ou 17. Esta fase instala o OpenJDK no sistema e coloca o `autoupgrade.jar` inicial em `${INSTALL_DIR}`, pronto para a Atividade 2.

### 8.1 — Instalar OpenJDK 17

```bash
${SSH} "
dnf install -y java-17-openjdk java-17-openjdk-devel 2>&1 | tail -5
java -version
"
```
Verificar: deve retornar `openjdk version \"17...\"`.

Se `java-17-openjdk` não estiver disponível no repositório, usar OpenJDK 11:
```bash
${SSH} "dnf install -y java-11-openjdk java-11-openjdk-devel 2>&1 | tail -5 && java -version"
```

### 8.2 — Definir JAVA_HOME no profile do oracle

**Nota (2026-08-13):** a versão anterior deste passo (escaping aninhado `\$`/`\\\$` dentro
de uma única string `${SSH} "..."`) corrompeu o `.bash_profile` do oracle nesta sessão — a
linha `export PATH=...` acabou recebendo o `$PATH` **local do Windows/Git-Bash** (cheio de
`/c/Program Files/...`, `/mingw64/...`) em vez do `$PATH` remoto literal. Escaping de 3+
níveis (local bash → argumento do ssh → bash remoto) é frágil demais pra confiar. Usar em
vez disso `ssh ... 'bash -s' <<'REMOTE_EOF' ... REMOTE_EOF` — heredoc local com delimitador
**entre aspas simples**, que não expande nada localmente, então tudo dentro é enviado
literal para o bash remoto interpretar:

```bash
ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR root@${VM_IP} 'bash -s' <<'REMOTE_EOF'
JAVA_BIN=$(alternatives --list 2>/dev/null | grep java | awk '{print $3}' | head -1)
JAVA_HOME_PATH=$(dirname $(dirname $JAVA_BIN))
echo "JAVA_HOME detectado: $JAVA_HOME_PATH"

grep -q 'JAVA_HOME' /home/oracle/.bash_profile || {
  echo "export JAVA_HOME=$JAVA_HOME_PATH" >> /home/oracle/.bash_profile
  echo 'export PATH=$JAVA_HOME/bin:$PATH' >> /home/oracle/.bash_profile
}
echo 'JAVA_HOME adicionado ao .bash_profile do oracle'
REMOTE_EOF
```
Duas linhas de `echo` separadas em vez de uma só multi-linha — a segunda usa aspas
**simples** (`'...'`) porque o `$JAVA_HOME`/`$PATH` ali devem ficar literais no arquivo
(para serem expandidos só no login futuro do oracle), enquanto a primeira usa aspas duplas
de propósito para expandir `$JAVA_HOME_PATH` **agora** (valor real detectado nesta run). Note
que `${ORACLE_USER}`/`${VM_IP}`/`${SSH_KEY}` continuam sendo substituídos pelo agente antes
de rodar — só o conteúdo *dentro* do heredoc `<<'REMOTE_EOF'` fica livre de expansão local.
```

### 8.3 — Copiar autoupgrade.jar do Oracle Home para /install

O `autoupgrade.jar` que acompanha o Oracle 19c é o ponto de partida. A Atividade 2 irá atualizá-lo para a versão mais recente.

```bash
${SSH} "
SRC=${ORACLE_HOME_19C}/rdbms/admin/autoupgrade.jar

if [ -f \"\$SRC\" ]; then
  cp \"\$SRC\" ${AUTOUPGRADE_JAR}
  chown ${ORACLE_USER}:${ORACLE_GROUP} ${AUTOUPGRADE_JAR}
  echo 'autoupgrade.jar copiado para ${INSTALL_DIR}'
  java -jar ${AUTOUPGRADE_JAR} -version 2>/dev/null | head -3
else
  echo 'ERRO: autoupgrade.jar nao encontrado em '\$SRC
  echo 'Verifique se o Oracle Home foi instalado corretamente na Fase 3'
fi
"
```

### 8.4 — Criar estrutura de diretórios do AutoUpgrade em /install

```bash
${SSH} "
mkdir -p ${INSTALL_DIR}
mkdir -p ${PATCH_DIR}
mkdir -p ${AUTOUPGRADE_LOG_DIR}
chown -R ${ORACLE_USER}:${ORACLE_GROUP} ${INSTALL_DIR} ${AUTOUPGRADE_LOG_DIR}
chmod -R 775 ${INSTALL_DIR}
echo '=== Estrutura /install ==='
ls -la ${INSTALL_DIR}
"
```

---

## Verificação Final da Atividade 1

```bash
${SSH} "
echo '=== Hostname ==='
hostname -f
echo '=== Mounts ==='
df -h /u01 /u02 2>/dev/null
echo '=== Oracle processes ==='
ps aux | grep -E 'pmon|lsnr' | grep -v grep
echo '=== Listener status ==='
su - ${ORACLE_USER} -c 'export ORACLE_HOME=${ORACLE_HOME_19C}; export PATH=\$ORACLE_HOME/bin:\$PATH; lsnrctl status' 2>/dev/null | grep -E 'Version|Status|Listening'
echo '=== Database open mode ==='
su - ${ORACLE_USER} -c 'export ORACLE_HOME=${ORACLE_HOME_19C}; export ORACLE_SID=${ORACLE_SID}; export PATH=\$ORACLE_HOME/bin:\$PATH; echo \"select name, open_mode, cdb from v\\\$database;\" | sqlplus -s / as sysdba' 2>/dev/null
"
```

**Atividade 1 concluída com sucesso quando:**
- `/u02` montado e visível em `df -h`
- Processos `pmon_ORCL` e `tnslsnr` ativos
- `OPEN_MODE=READ WRITE` e `CDB=NO`
- `java -version` retorna OpenJDK 17 (ou 11)
- `${INSTALL_DIR}/autoupgrade.jar` presente e funcional
