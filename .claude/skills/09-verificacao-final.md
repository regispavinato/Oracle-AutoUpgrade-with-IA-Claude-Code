---
description: Atividade 9 — Verificação final do ambiente Oracle 23ai e publicação de resumo no LinkedIn
---

# Atividade 9 — Verificação Final + Resumo para LinkedIn

## Contexto
Validação completa do ambiente após todas as atividades da imersão, seguida da geração de um post profissional para publicação no LinkedIn documentando a jornada técnica.

## Carregar variáveis

Leia `.env` com o Read tool. Nunca exiba senhas.

```bash
SSH="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 oracle@${VM_IP}"
SSH_ROOT="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 root@${VM_IP}"
# SID após Atividade 5
REAL_SID=CDBORCL
```

---

## FASE 1 — Health Check: Instância e Containers

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

echo '--- Instancia ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 40 LINESIZE 120 FEEDBACK OFF
COL instance_name FORMAT A16
COL version_full  FORMAT A18
COL status        FORMAT A10
COL host_name     FORMAT A30
SELECT instance_name, version_full, status, TO_CHAR(startup_time,'DD/MM/YYYY HH24:MI') startup, host_name FROM v\$instance;
ENDSQL

echo '--- Database ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 10 LINESIZE 100 FEEDBACK OFF
SELECT name, cdb, open_mode, log_mode FROM v\$database;
ENDSQL

echo '--- PDBs ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 20 LINESIZE 120 FEEDBACK OFF
COL name FORMAT A12
COL open_mode FORMAT A12
COL version_full FORMAT A18
SELECT p.con_id, p.name, p.open_mode, r.version_full
FROM v\$pdbs p
LEFT JOIN cdb_registry r ON r.con_id = p.con_id AND r.comp_id = 'CATPROC'
ORDER BY p.con_id;
ENDSQL
"
```

---

## FASE 2 — Health Check: Componentes do Dicionário

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

echo '--- Componentes nao-VALID (esperado: 0 linhas ou apenas OPTION OFF) ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 40 LINESIZE 130 FEEDBACK OFF
COL comp_name FORMAT A45
COL version_full FORMAT A16
COL status FORMAT A12
SELECT con_id, comp_name, version_full, status FROM cdb_registry WHERE status NOT IN ('VALID','OPTION OFF') ORDER BY con_id, comp_name;
ENDSQL

echo '--- Contagem por status ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 20 LINESIZE 80 FEEDBACK OFF
SELECT con_id, status, count(*) qtd FROM cdb_registry GROUP BY con_id, status ORDER BY con_id, status;
ENDSQL

echo '--- Objetos invalidos por container ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 20 LINESIZE 80 FEEDBACK OFF
SELECT con_id, count(*) invalid_objects FROM cdb_objects WHERE status='INVALID' GROUP BY con_id ORDER BY con_id;
ENDSQL

echo '--- Patches aplicados (ultimos 6) ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 20 LINESIZE 140 FEEDBACK OFF
COL description FORMAT A55
SELECT con_id, patch_id, status, TO_CHAR(action_time,'DD/MM/YYYY HH24:MI') applied_at, description FROM cdb_registry_sqlpatch ORDER BY action_time DESC FETCH FIRST 6 ROWS ONLY;
ENDSQL
"
```

---

## FASE 3 — Health Check: Memória e Armazenamento

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

echo '--- SGA ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 20 LINESIZE 100 FEEDBACK OFF
COL name FORMAT A35
SELECT name, ROUND(value/1048576,0) mb FROM v\$sga ORDER BY value DESC;
ENDSQL

sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 5 LINESIZE 60 FEEDBACK OFF
SELECT ROUND(SUM(value)/1048576,0) sga_total_mb FROM v\$sga;
ENDSQL

echo '--- PGA ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 5 LINESIZE 60 FEEDBACK OFF
SELECT ROUND(value/1048576,0) pga_target_mb FROM v\$parameter WHERE name='pga_aggregate_target';
ENDSQL

echo '--- PROCESSES e SESSIONS ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 10 LINESIZE 80 FEEDBACK OFF
SELECT name, value FROM v\$parameter WHERE name IN ('processes','sessions') ORDER BY name;
ENDSQL

echo '--- FRA ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 10 LINESIZE 120 FEEDBACK OFF
COL name FORMAT A30
SELECT name, ROUND(space_limit/1073741824,1) limit_gb, ROUND(space_used/1073741824,1) used_gb, ROUND(space_reclaimable/1073741824,1) reclaimable_gb, ROUND(space_used/space_limit*100,1) pct_used FROM v\$recovery_file_dest;
ENDSQL

echo '--- Datafiles ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 10 LINESIZE 80 FEEDBACK OFF
SELECT con_id, count(*) datafiles, ROUND(SUM(bytes)/1073741824,2) size_gb FROM cdb_data_files GROUP BY con_id ORDER BY con_id;
ENDSQL

echo '--- Redo Logs ---'
sqlplus -s / as sysdba <<'ENDSQL'
SET PAGESIZE 20 LINESIZE 100 FEEDBACK OFF
SELECT l.group#, l.members, l.status, ROUND(f.bytes/1048576,0) size_mb FROM v\$log l JOIN v\$logfile f ON f.group#=l.group# ORDER BY l.group#;
ENDSQL
"
```

---

## FASE 4 — Health Check: Alert Log (últimas 24h)

```bash
${SSH} "
# Verificar erros no alert log das últimas 24 horas
ALERT_LOG=\$(find /u01/app/oracle/diag/rdbms/cdborcl/CDBORCL/trace -name 'alert_CDBORCL.log' 2>/dev/null | head -1)

echo '=== ERROS no alert log (últimas 24h) ==='
if [ -f \"\$ALERT_LOG\" ]; then
  awk -v cutoff=\"\$(date -d '24 hours ago' '+%Y-%m-%dT%H')\" '
    /^[0-9]{4}-[0-9]{2}-[0-9]{2}T/ { current_ts = \$1 }
    current_ts >= cutoff && /ORA-|ERROR|FATAL|CORRUPT/ { print current_ts, \$0 }
  ' \"\$ALERT_LOG\" | grep -v 'ORA-32004\|ORA-12751' | head -20
  echo '(ORA-32004=parâmetro obsoleto benigno; ORA-12751=Resource Manager — já corrigido)'
else
  echo 'Alert log não encontrado'
fi

echo ''
echo '=== Últimas 10 linhas do alert log ==='
tail -10 \"\$ALERT_LOG\" 2>/dev/null
"
```

---

## FASE 5 — Health Check: Listener e Conectividade

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export PATH=\$ORACLE_HOME/bin:\$PATH

echo '=== Listener Status ==='
lsnrctl status 2>/dev/null | grep -E 'Version TNSLSNR|Uptime|Services Summary|Service|Instance|STATUS'
"
```

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH

echo '=== Teste de conexão via listener ==='
sqlplus -s sys/${ORACLE_PWD}@localhost/CDBORCL as sysdba << 'EOF'
SELECT 'CDB OK - ' || instance_name || ' v' || version_full AS status FROM v\$instance;
EOF

sqlplus -s sys/${ORACLE_PWD}@localhost/orclpdb as sysdba << 'EOF'
SELECT 'ORCLPDB OK - ' || name AS status FROM v\$database;
EOF
"
```

---

## FASE 6 — Health Check: oratab, bash_profile e systemd

```bash
${SSH_ROOT} "
echo '=== /etc/oratab ==='
grep -v '^#' /etc/oratab | grep -v '^$'

echo ''
echo '=== oracle bash_profile ==='
grep -E 'ORACLE_HOME|ORACLE_SID|PATH' /home/oracle/.bash_profile | grep -v '^#'

echo ''
echo '=== systemd oracle services ==='
for svc in oracle-database oracle-listener; do
  if systemctl list-unit-files \${svc}.service &>/dev/null; then
    echo \"\$svc: \$(systemctl is-active \${svc}.service 2>/dev/null || echo 'not-found')\"
  fi
done

echo ''
echo '=== Oracle Home ativo ==='
ls -la \$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)/bin/oracle 2>/dev/null || echo 'AVISO: oratab pode estar desatualizado'
"
```

---

## FASE 6.5 — Health Check: Rotina de Backup RMAN (Atividade 8)

```bash
${SSH} "
echo '=== crontab do oracle ==='
crontab -l 2>/dev/null | grep rman_backup || echo 'AVISO: cron de backup nao encontrado'

echo ''
echo '=== ultimas execucoes ==='
tail -5 /backup/logs/cron.log 2>/dev/null || echo 'sem execucoes registradas ainda'

echo ''
echo '=== conteudo /backup ==='
ls -lh /backup/*.ctl /backup/autobackup_* /backup/snapcf_* 2>/dev/null
du -sh /backup 2>/dev/null
"
```

```bash
${SSH} "
export ORACLE_HOME=${ORACLE_HOME_23AI}
export ORACLE_SID=CDBORCL
export PATH=\$ORACLE_HOME/bin:\$PATH
rman target / << 'EOF'
LIST BACKUP SUMMARY;
EOF
"
```

---

## FASE 7 — Sumário Final e Post LinkedIn

Após coletar todos os resultados acima, gerar o seguinte output consolidado.

### 7.1 — Sumário do ambiente (exibir no terminal)

Consolidar os dados coletados e exibir:

```
╔══════════════════════════════════════════════════════════════╗
║          ORACLE DATABASE 23ai — AMBIENTE VALIDADO            ║
╠══════════════════════════════════════════════════════════════╣
║  Versão        : 23.26.x.x.x (23ai)                        ║
║  SID/DB Name   : CDBORCL                                    ║
║  Host          : srv19c-2.localdomain                       ║
║  Oracle Home   : /u01/app/oracle/product/23.0.0/dbhome_1   ║
╠══════════════════════════════════════════════════════════════╣
║  CONTAINERS                                                  ║
║  • CDB$ROOT     READ WRITE  23.26.x  ✓                     ║
║  • PDB$SEED     READ ONLY   23.26.x  ✓                     ║
║  • ORCLPDB      READ WRITE  23.26.x  ✓                     ║
╠══════════════════════════════════════════════════════════════╣
║  MEMÓRIA                                                     ║
║  • SGA Total   : X MB                                       ║
║  • PGA Target  : X MB                                       ║
╠══════════════════════════════════════════════════════════════╣
║  ARMAZENAMENTO                                               ║
║  • FRA         : X.x GB / X GB (X%)                        ║
║  • Datafiles   : X GB total                                 ║
╠══════════════════════════════════════════════════════════════╣
║  BACKUP                                                      ║
║  • Rotina RMAN : FULL domingo + incremental diário (16:00)  ║
║  • Retenção    : recovery window de 1 dia                   ║
╠══════════════════════════════════════════════════════════════╣
║  SAÚDE                                                       ║
║  • Componentes inválidos : 0                                ║
║  • Objetos inválidos     : 0                                ║
║  • Erros no alert log    : 0 (últimas 24h)                  ║
╚══════════════════════════════════════════════════════════════╝
```

Preencher com os valores reais coletados nas fases anteriores.

### 7.2 — Post para LinkedIn

Gerar o post abaixo preenchido com os dados reais do ambiente. O texto deve ser em português, tom profissional e técnico, máximo 4 páginas impressas (≈ 3.000 caracteres no LinkedIn).

---

**[POST LINKEDIN — gerado pela skill, preencher com dados reais]**

```
🚀 Imersão Oracle AutoUpgrade — Do Zero ao Oracle 23ai em 2 dias

Concluí uma imersão técnica intensiva cobrindo o ciclo completo de 
modernização de bancos Oracle usando a ferramenta AutoUpgrade.

🔹 O que foi feito (9 atividades):

1️⃣  Preparação do ambiente
    • Oracle Linux 8.10 | AMD Ryzen 7 | 6 vCPUs | 6 GB RAM
    • LVM (/u01 = 100 GB | /u02 = 40 GB com XFS)
    • Instalação Oracle Database 19c via Gold Image
    • DBCA silent, Listener, configuração de serviços systemd

2️⃣  Download do último patch via AutoUpgrade
    • java -jar autoupgrade.jar -download
    • Atualização automática para a versão mais recente do jar

3️⃣  Criação de Oracle Home out-of-place
    • Novo home isolado para aplicação de patch sem impacto no banco ativo
    • Separação clara entre binary home e dados

4️⃣  Patching 19c com AutoUpgrade (out-of-place)
    • Release Update aplicado via OPatch no novo home
    • AutoUpgrade modo deploy: shutdown → troca de home → datapatch → startup
    • Banco migrado com zero intervenção manual no dicionário

5️⃣  Conversão non-CDB → Multitenant
    • AutoUpgrade modo noncdbtopdb
    • CDB CDBORCL criado via DBCA silent
    • ORCL convertido para ORCLPDB em [X] minutos
    • Desafio: AutoUpgrade 26.x não suporta mais target_cdb=NEW
    • OMF ativado e datafiles reorganizados dentro do diretório do CDB

6️⃣  Migração 19c → Oracle 23ai (23.26.1)
    • Upgrade completo do dicionário via catctl.pl
    • CDB$ROOT + PDB$SEED + ORCLPDB — tudo em [1h17m]
    • Desafio resolvido: bug de deadlock Java contornado com
      catctl.pl direto + Resource Manager desabilitado para utlrp

7️⃣  Patch 23.26.1 → 23.26.2
    • Patching out-of-place no dbhome_2
    • AutoUpgrade modo deploy com -noconsole + nohup

8️⃣  Configuração de backup RMAN
    • FULL aos domingos + incremental no resto da semana
    • Disparo via cron às 16:00, retenção de 1 dia
    • Controlfile autobackup + snapshot controlfile dedicados

9️⃣  Verificação final
    • 0 componentes inválidos | 0 objetos inválidos | 0 erros no alert log
    • SGA: [X] MB | PGA: [X] MB | FRA: [X] GB
    • Banco 100% funcional em Oracle 23ai, com backup automatizado

─────────────────────────────────────────────

🧠 Principais aprendizados técnicos:

✅ AutoUpgrade 26.x mudou o parâmetro de log dir:
   global.autoupg_log_dir → global.global_log_dir (em alguns modos)

✅ target_cdb=NEW foi descontinuado — o CDB alvo deve existir

✅ catupgrd.sql desde o 12.2 só pode ser invocado via catctl.pl
   (catcon.pl e chamada direta pelo SQL*Plus falham)

✅ Resource Manager pode matar parallel slaves do utlrp com ORA-12751
   → sempre desabilitar antes: ALTER SYSTEM SET resource_manager_plan=''

✅ FRA precisa de ≥ 20 GB antes de qualquer upgrade
   (catupgrd gera 10-15 GB de redo em VM pequena)

✅ sqlplus -prelim permite SHUTDOWN ABORT quando
   a instância não aceita conexões (ROW CACHE LOCK)

✅ AutoUpgrade modo deploy sempre com -noconsole + nohup
   para evitar deadlock de pipe stdin e perda de SSH

✅ noncdbtopdb não move datafiles — ativar OMF e reorganizar
   os arquivos da PDB dentro do diretório do CDB é passo extra

─────────────────────────────────────────────

📊 Ambiente final validado:

  Oracle Database 23ai (23.26.x)
  CDB: CDBORCL  |  PDB: ORCLPDB
  Versão: [VERSION_FULL]
  Patch: [PATCH_ID] — [PATCH_DESC]
  Componentes: todos VALID
  Objetos inválidos: 0
  Erros no alert log (24h): 0
  Backup: RMAN automatizado (FULL semanal + incremental diário)

─────────────────────────────────────────────

🛠️  Stack técnica:
Oracle Database 23ai • AutoUpgrade • Oracle Linux 8 • LVM
Multitenant/CDB • OPatch • RMAN • catctl.pl • systemd

#OracleDatabase #Oracle23ai #AutoUpgrade #DBA #Multitenant
#OracleLinux #DatabaseUpgrade #DevOps #CloudMigration
#OracleCertified #DatabaseAdministration
```

---

**Instrução para Claude:** Preencher os campos entre colchetes `[...]` com os valores reais coletados nas fases 1-6.5 antes de exibir o post. Verificar se há erros críticos no alert log — se houver, listar no sumário e NÃO marcar o ambiente como "100% funcional" sem que sejam explicados.
