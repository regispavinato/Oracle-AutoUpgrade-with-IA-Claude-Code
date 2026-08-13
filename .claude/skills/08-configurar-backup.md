---
description: Atividade 8 — Configurar rotina de backup RMAN (FULL semanal + incremental diário) via cron
---

# Atividade 8 — Configuração de Backup RMAN

## Contexto
Com o banco `CDBORCL` já em 23.26.2 (Atividade 7), esta atividade configura uma rotina de backup RMAN recorrente: **FULL (nível 0) aos domingos**, **incremental (nível 1) no resto da semana**, disparada via **cron às 16:00**, com destino `/backup` e **retenção de recovery window de 1 dia**. Inclui backup do controlfile e realoca o snapshot controlfile para dentro do diretório de backup.

## Pré-requisitos
- Atividade 7 concluída: banco `CDBORCL` rodando (verificar home atual via `/etc/oratab`, não assumir `${ORACLE_HOME_23AI}` do `.env` — pode estar desatualizado após patches)
- Espaço em disco livre suficiente para o backup em `/backup` (ver Fase 0)

## Carregar variáveis

Leia `.env` com o Read tool. Nunca exiba senhas.

```bash
SSH="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 oracle@${VM_IP}"
SSH_ROOT="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 root@${VM_IP}"

BACKUP_DIR=/backup
```

---

## FASE 0 — Pré-verificações

### 0.1 — Descobrir o Oracle Home atual (via oratab, nunca fixo)

```bash
${SSH} "grep '^CDBORCL:' /etc/oratab"
```

### 0.2 — Verificar espaço em disco disponível

Escolher o filesystem com mais espaço livre entre `/` e `/u02` para hospedar `/backup` (não assumir sempre o mesmo).

```bash
${SSH} "df -h / /u02"
```

---

## FASE 1 — Criar diretório de backup

```bash
${SSH_ROOT} "
mkdir -p ${BACKUP_DIR}/logs
chown -R ${ORACLE_USER}:${ORACLE_GROUP} ${BACKUP_DIR}
chmod 750 ${BACKUP_DIR}
"
```

---

## FASE 2 — Configurar parâmetros persistentes do RMAN

Exibir para o usuário verificar antes de aplicar:

```
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 1 DAYS;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '${BACKUP_DIR}/autobackup_%F';
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '${BACKUP_DIR}/%U';
CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO COMPRESSED BACKUPSET;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '${BACKUP_DIR}/snapcf_CDBORCL.f';
```

> `COMPRESSED BACKUPSET` é o padrão adotado por ser uma VM com disco pequeno — se o ambiente tiver disco folgado, pode-se usar `BACKUPSET` sem compressão para reduzir CPU no backup.

```bash
${SSH} "
ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_HOME
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
rman target / << 'EOF'
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 1 DAYS;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '${BACKUP_DIR}/autobackup_%F';
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '${BACKUP_DIR}/%U';
CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO COMPRESSED BACKUPSET;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '${BACKUP_DIR}/snapcf_CDBORCL.f';
SHOW ALL;
EOF
"
```

---

## FASE 3 — Criar script de backup

O script decide o nível do backup pelo dia da semana (`date +%u`: 7=domingo=FULL/nível 0, resto=incremental/nível 1) e sempre inclui backup do controlfile, limpeza de obsoletos e crosscheck.

```bash
${SSH} "mkdir -p /u01/app/oracle/scripts"
```

```bash
${SSH} "
cat > /u01/app/oracle/scripts/rman_backup_cdborcl.sh << 'SCRIPTEOF'
#!/bin/bash
#############################################################
# RMAN Backup CDBORCL - FULL (nivel 0) aos domingos,
# INCREMENTAL (nivel 1) no resto da semana.
# Disparado via cron as 16:00. Retencao: recovery window de 1 dia.
#############################################################

export ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

BACKUP_DIR=${BACKUP_DIR}
LOG_DIR=\${BACKUP_DIR}/logs
TIMESTAMP=\$(date +%Y%m%d_%H%M%S)
DOW=\$(date +%u)   # 1=segunda ... 7=domingo

if [ \"\$DOW\" -eq 7 ]; then
  LEVEL=0
  LABEL=FULL
else
  LEVEL=1
  LABEL=INCR
fi

LOG_FILE=\"\${LOG_DIR}/rman_\${LABEL}_\${TIMESTAMP}.log\"

rman target / log=\"\${LOG_FILE}\" << EOF
BACKUP INCREMENTAL LEVEL \${LEVEL} DATABASE PLUS ARCHIVELOG DELETE INPUT;
BACKUP CURRENT CONTROLFILE;
DELETE NOPROMPT OBSOLETE;
CROSSCHECK BACKUP;
DELETE NOPROMPT EXPIRED BACKUP;
EOF

RC=\$?
echo \"\$(date '+%Y-%m-%d %H:%M:%S') - Backup \${LABEL} (level \${LEVEL}) finalizado com RC=\${RC}. Log: \${LOG_FILE}\" >> \"\${LOG_DIR}/cron.log\"
exit \$RC
SCRIPTEOF
chmod 750 /u01/app/oracle/scripts/rman_backup_cdborcl.sh
cat /u01/app/oracle/scripts/rman_backup_cdborcl.sh
"
```

> **Nota de escaping:** o heredoc externo (`cat > ... << 'SCRIPTEOF'`) escreve o conteúdo literalmente (delimitador entre aspas simples = sem expansão no lado remoto). Mas todo esse bloco é, primeiro, uma string entre aspas duplas na camada LOCAL (`${SSH} "..."`) — então cada `$` que precisa sobreviver e virar `$` literal dentro do arquivo remoto precisa de **uma única barra** (`\$`), e cada `"` literal precisa de `\"`. Não usar duas barras — isso é o padrão já validado (testado e confirmado com `cat` do arquivo final em 2026-07-19). O heredoc interno do próprio script (`rman target / log="..." << EOF ... EOF`, sem aspas no delimitador) é o que garante que `${LEVEL}` etc. sejam expandidos de verdade quando o script rodar (não durante a escrita do arquivo).

---

## FASE 4 — Testar manualmente antes de agendar

```bash
${SSH} "/u01/app/oracle/scripts/rman_backup_cdborcl.sh; echo \"EXIT_CODE=\$?\""
```

Se a execução demorar mais que alguns minutos (comum se houver muitos archivelogs acumulados), rodar em background e monitorar via poll:

```bash
${SSH} "nohup /u01/app/oracle/scripts/rman_backup_cdborcl.sh > /tmp/rman_test.out 2>&1 &"
```

```bash
${SSH} "
until ! pgrep -f 'rman_backup_cdborcl.sh' >/dev/null 2>&1; do sleep 5; done
tail -5 ${BACKUP_DIR}/logs/cron.log
"
```

Verificar o resultado:

```bash
${SSH} "
ls -lh ${BACKUP_DIR}/
du -sh ${BACKUP_DIR}
"
```

```bash
${SSH} "
ORACLE_HOME=\$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_HOME
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
rman target / << 'EOF'
LIST BACKUP SUMMARY;
REPORT SCHEMA;
EOF
"
```

Confirmar: pelo menos uma controlfile autobackup (`autobackup_c-*`) e o `snapcf_CDBORCL.f` fisicamente em `${BACKUP_DIR}`.

---

## FASE 5 — Instalar o cron job

```bash
${SSH} "
(crontab -l 2>/dev/null; echo '0 16 * * * /u01/app/oracle/scripts/rman_backup_cdborcl.sh') | crontab -
crontab -l
"
```

Confirmar que o `crond` está ativo:

```bash
${SSH_ROOT} "systemctl is-active crond; systemctl is-enabled crond"
```

---

## FASE 6 — Verificação final

```bash
${SSH} "crontab -l | grep rman_backup"
${SSH} "cat ${BACKUP_DIR}/logs/cron.log"
```

---

## Resultado esperado

- `/backup` criado, dono `oracle`, com `logs/` para os logs de cada execução
- RMAN configurado com retenção `RECOVERY WINDOW OF 1 DAYS`, canal padrão e controlfile autobackup apontando para `/backup`, snapshot controlfile também em `/backup`
- Script `/u01/app/oracle/scripts/rman_backup_cdborcl.sh` funcional, testado manualmente com sucesso (RC=0)
- Cron do usuário `oracle`: `0 16 * * * /u01/app/oracle/scripts/rman_backup_cdborcl.sh`
- Backup FULL (domingo) ou incremental (demais dias) rodando automaticamente todo dia às 16:00, com limpeza automática de obsoletos/expirados a cada execução
