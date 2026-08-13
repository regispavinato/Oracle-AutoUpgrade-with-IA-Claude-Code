# Oracle Imersão AutoUpgrade — Contexto do Projeto

## Objetivo
Imersão de dois dias sobre Oracle AutoUpgrade cobrindo instalação, patching, conversão para Multitenant e migração para 23ai.

## Estrutura de Skills
As skills estão em `.claude/skills/` e seguem numeração das atividades da imersão:

| Skill | Atividade |
|---|---|
| `oracle-imersao` | **Principal** — roteador, verifica pré-requisitos e invoca skills específicas |
| `00-criar-vm` | Criar VM no VirtualBox (100GB+40GB, 6GB RAM, 6 vCPU, bridge+IP estático) + instalar Oracle Linux 8.10 Server (sem GUI) via kickstart |
| `01-prepare-ambiente` | OS prep + LVM + Install Oracle 19c + DBCA + Listener + startup |
| `02-download-patch` | Download do último patch via autoupgrade |
| `03-criar-oracle-home` | Criar novo Oracle Home com autoupgrade |
| `04-aplicar-patch` | Aplicar patch com autoupgrade |
| `05-converter-multitenant` | Converter non-CDB para Multitenant com autoupgrade |
| `06-migrar-23ai` | Migrar 19c → 23.26.1 com autoupgrade |
| `07-patch-23262` | Aplicar patch 23.26.1 → 23.26.2 com autoupgrade |
| `08-configurar-backup` | Configurar rotina de backup RMAN (FULL semanal + incremental via cron) |
| `09-verificacao-final` | Verificação final do ambiente + resumo LinkedIn |

## Responsabilidades da IA neste projeto

### Arquivos de configuração do AutoUpgrade (.cfg)
- **Claude cria TODOS os arquivos `.cfg`** do AutoUpgrade — o usuário não os fornece.
- Cada skill de patching/upgrade/conversão gera seu próprio `.cfg` com a sintaxe correta antes de executar.
- Todos os `.cfg` são salvos em `${INSTALL_DIR}` (valor de `INSTALL_DIR` no `.env`).
- Antes de criar um `.cfg`, exibir o conteúdo completo para o usuário verificar.

### autoupgrade.jar
- O `autoupgrade.jar` inicial é copiado de `${ORACLE_HOME_19C}/rdbms/admin/autoupgrade.jar` para `${INSTALL_DIR}/` na **Fase 8 da Atividade 1**.
- A Atividade 2 executa `java -jar autoupgrade.jar -download` para atualizar para a versão mais recente.
- O usuário nunca precisa baixar ou copiar o `autoupgrade.jar` manualmente.

### Java (OpenJDK)
- O OpenJDK 17 é instalado pela **Fase 8 da Atividade 1** via `dnf` para uso geral do SO.
- Usar **sempre** o JDK embutido no Oracle Home (`${ORACLE_HOME}/jdk/bin/java`), nunca o
  `java` do sistema, para executar `autoupgrade.jar`:
  - Skills 02, 04: `${ORACLE_HOME_19C}/jdk/bin/java`
  - Skills 06, 07: `${ORACLE_HOME_23AI}/jdk/bin/java`
- Cada skill define a variável `JAVA` apontando para o JDK correto do Oracle Home.
- **Correção (2026-08-13, testado ponta-a-ponta na Atividade 4):** o JDK embutido no Oracle
  Home 19c — base e depois do RU aplicado — é **Java 8** (`1.8.0_201` → `1.8.0_481` após o
  patch), não Java 11 como este documento afirmava antes. Mesmo assim, `autoupgrade.jar`
  26.5.260807 rodou sem erro nenhum com esse JDK 8 em `-version`, `-mode analyze` e
  `-mode deploy` completo (datapatch + utlrp inclusos, `Jobs failed [0]`) — o alerta de
  "Unsupported Java Runtime Environment" não se confirmou para os Oracle Homes 19c. Não foi
  testado rodar com o Java 17 do sistema (a orientação de sempre usar o JDK embutido
  continua válida e segura, só a premissa do porquê estava errada). O JDK do
  `${ORACLE_HOME_23AI}` (skills 06/07) ainda não foi verificado nesta sessão — pode
  realmente ser Java 11/17, já que o 23ai é uma base bem mais nova que o 19.3.

### Arquivos de configuração gerados por skill

| Skill | Arquivo .cfg criado em /install |
|---|---|
| `02-download-patch` | `autoupgrade_analyze.cfg` |
| `04-aplicar-patch` | `autoupgrade_patch.cfg` |
| `05-converter-multitenant` | `autoupgrade_convert.cfg` |
| `06-migrar-23ai` | `autoupgrade_upgrade23ai.cfg` |
| `07-patch-23262` | `autoupgrade_patch23262.cfg` |
| `08-configurar-backup` | *(sem .cfg — configuração RMAN via `CONFIGURE`, não AutoUpgrade)* |
| `09-verificacao-final` | *(sem .cfg — apenas queries de verificação)* |

---

## Regras de Segurança (SEMPRE seguir)
1. **Nunca** escrever senhas, IPs ou hostnames em skill files ou logs visíveis.
2. **Sempre** carregar variáveis sensíveis do arquivo `.env` via Read tool antes de usar.
3. **Nunca** exibir o conteúdo do `.env` na resposta — usar apenas os valores, mascarados em logs.
4. Ao logar comandos no output, mascarar senhas com `***`.
5. SSH via chave (`SSH_KEY` do `.env`); nunca passar senha via argumento `-pw` em linha visível.

## Transição de ORACLE_SID ao longo das atividades

| Skills | ORACLE_SID | Banco |
|--------|-----------|-------|
| 01 → 04 | `ORCL` (do `.env`) | non-CDB |
| 05 | transição — começa com `ORCL`, termina com `CDBORCL` | non-CDB → CDB |
| 06 → 09 | `CDBORCL` (**ignorar** `ORACLE_SID` do `.env`) | CDB com PDB ORCLPDB |

> O arquivo `.env` mantém `ORACLE_SID=ORCL` por razão histórica. Nas skills 06-09, **nunca** usar `${ORACLE_SID}` do `.env` — usar sempre `CDBORCL` explicitamente.

## Pré-requisitos para usar as skills
- Arquivo `.env` preenchido (copiar de `.env.template`).
- SSH key configurada entre esta máquina e a VM (o skill `oracle-imersao` oferece configurar isso).
- Acesso root ou sudo na VM via SSH.
- Gold Images copiadas para `/install` na VM antes de rodar as skills de instalação/upgrade.

## Regras Operacionais Aprendidas (aplicar em toda skill de patch/upgrade)

### Gestão de memória em VM pequena (RAM restrita)
- **Nunca rodar dois bancos Oracle ativos simultaneamente sem necessidade.** Antes de criar um novo banco (DBCA) com outro banco já em pé, parar o banco existente (`shutdown immediate`) para liberar RAM primeiro — não como correção reativa a uma falha, mas como passo padrão.
- Causa raiz confirmada (Atividade 5): a limitação é de **RAM física do SO**, não de SGA individual mal dimensionada — a soma das SGAs/PGAs de instâncias concorrentes mais o overhead do SO é o que estoura, gerando swap pesado que pode derrubar o `utlrp` do DBCA (que sobe múltiplos parallel slaves).
- **Nunca fazer `shutdown`/bounce de uma instância enquanto um `dbca`, `opatch apply` ou AutoUpgrade em background ainda pode estar rodando.** Se esses processos falharem sozinhos, fazem seu próprio rollback; um shutdown prematuro no meio disso pode parecer ter causado corrupção (ex. `ORA-00205` por control files ausentes) quando na verdade o processo já tinha desfeito a criação. Sempre aguardar o processo encerrar sozinho (poll via `pgrep`) antes de tocar na instância.

### Limpeza preventiva de jobs do AutoUpgrade
- Antes de qualquer `-mode analyze` ou `-mode deploy`, verificar se há job anterior pendente para o mesmo SID (mesmo se de uma atividade/config diferente) e limpar proativamente com `-clear_recovery_data -jobs <N>` — não esperar o erro "unfinished execution" aparecer para só então reagir.
- Como descobrir o job mais recente: listar os diretórios em `${AUTOUPGRADE_LOG_DIR}/<SID>/<SID>/*` (ou `<log_dir>/<SID>/<SID>/*` do `.cfg` da atividade) ordenados por data e pegar o maior número; ou checar o `status.log` da execução anterior.
- Se `-clear_recovery_data -jobs <N>` retornar "not found", é inofensivo — seguir em frente.

### Handoff de processos manuais para o systemd
- Sempre que uma skill iniciar listener/banco **manualmente** (`lsnrctl start`, `startup`) fora do systemd, ao final da atividade **parar manualmente e subir via `systemctl start`** para o systemd assumir o controle do processo.
- Motivo: o systemd não rastreia processos que não iniciou. Um `systemctl restart` num serviço "inactive" (mesmo com o processo real rodando por fora) tenta só o `ExecStart`, que falha com "already started" (TNS-01106 no listener) porque a porta/lock já está em uso.

### PDBs — abrir e persistir estado (padrão obrigatório, sempre em conjunto)
- Toda vez que uma PDB for criada, convertida (`noncdbtopdb`) ou passar por upgrade/patch via AutoUpgrade, executar **sempre as duas ações juntas, no mesmo passo**, nunca uma sem a outra:
  ```sql
  ALTER PLUGGABLE DATABASE <pdb> OPEN;
  ALTER PLUGGABLE DATABASE <pdb> SAVE STATE;
  ```
- Motivo: `dbstart`/restart do systemd sobe o CDB$ROOT mas **não abre PDBs sozinho** — sem o `SAVE STATE`, a PDB fica `MOUNTED` após qualquer restart do banco.

### OMF e organização de datafiles da PDB (padrão obrigatório, desde 2026-07-19)
- A conversão `noncdbtopdb` (Atividade 5) **não move os datafiles** — a PDB resultante fica fisicamente no caminho do non-CDB original (`${ORADATA_DIR}/${ORACLE_SID}/*.dbf`), fora da estrutura `${ORADATA_DIR}/CDBORCL/`.
- Sempre, logo após a conversão (skill 05, Fase 6.5): ativar OMF (`ALTER SYSTEM SET db_create_file_dest='${ORADATA_DIR}/CDBORCL' SCOPE=BOTH` em CDB$ROOT e na PDB) e mover os datafiles/tempfile da PDB para `${ORADATA_DIR}/CDBORCL/<PDB_NAME>/` via `ALTER DATABASE MOVE DATAFILE` (online, banco continua `OPEN`). Tempfiles não são suportados por `MOVE DATAFILE` — usar `ALTER TABLESPACE TEMP ADD TEMPFILE` + `ALTER DATABASE TEMPFILE ... DROP INCLUDING DATAFILES`.
- Motivo: mantém a mesma convenção já usada pela `pdbseed` (subdiretório com o nome da PDB dentro de `${ORADATA_DIR}/CDBORCL/`) e evita arquivos "órfãos" fora da estrutura do CDB, que podem ser confundidos com lixo (ou o oposto — deixados por engano por parecerem não relacionados ao CDB).
- Depois de mover, confirmar via `v$datafile`/`v$tempfile`/`v$logfile`/`v$controlfile` que nada mais referencia `${ORADATA_DIR}/${ORACLE_SID}/` antes de remover o diretório antigo (restam ali só o control file e os redo logs do non-CDB original, nunca usados pelo CDB).

## Ambiente Alvo
- **OS**: Oracle Linux 8.10 Server (minimal, sem GUI)
- **CPU**: AMD Ryzen 7 | **RAM**: 6 GB | **vCPUs**: 6
- **Disco OS**: 100 GB LVM (`/u01`)
- **Disco Dados**: 40 GB LVM (`/u02`) — `DISK_DEVICE` no `.env`
- **Hypervisor**: VirtualBox (`VBoxManage`, no host Windows) — VM provisionada pela skill `00-criar-vm`
- **Rede**: adaptador bridge do host + IP estático (configurado via kickstart na skill `00-criar-vm`)

### Kickstart (`ks.cfg`) da skill `00-criar-vm`
- Gerado a cada execução em `${VMS_BASE_DIR}\${VM_HOSTNAME}\ks.cfg`, fora do repositório do projeto — nunca copiar para dentro do projeto nem versionar.
- Contém `ROOT_PWD` em texto plano (exigência do formato kickstart). Nunca exibir o conteúdo desse arquivo no output — se precisar mostrar para o usuário, mascarar a linha `rootpw` (`rootpw --plaintext ***`), mesma regra de mascaramento de senhas do restante do projeto.
