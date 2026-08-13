# Oracle Imersão AutoUpgrade — 2 Dias

## Agenda

### Dia 1 — 10h às 18h (intervalo 12h–13h30)
| Horário | Atividade |
|---|---|
| 10h00 | **Atividade 1** — Preparação do ambiente Oracle Linux, LVM, instalação Oracle 19c non-CDB |
| 12h00 | Almoço |
| 13h30 | **Atividade 2** — Download do último patch com AutoUpgrade |
| 14h30 | **Atividade 3** — Criação de novo Oracle Home com AutoUpgrade |
| 16h00 | **Atividade 4** — Aplicação de patch com AutoUpgrade |

### Dia 2 — 10h às 18h (intervalo 12h–13h30)
| Horário | Atividade |
|---|---|
| 10h00 | **Atividade 5** — Converter banco 19c non-CDB para Multitenant com AutoUpgrade |
| 12h00 | Almoço |
| 13h30 | **Atividade 6** — Migrar banco 19c para Oracle 23ai (23.26.1) com AutoUpgrade |
| 15h30 | **Atividade 7** — Aplicar patch 23.26.1 → 23.26.2 com AutoUpgrade |
| 16h30 | **Atividade 8** — Patch em Oracle Restart 23ai |

---

## Ambiente

| Componente | Detalhe |
|---|---|
| SO | Oracle Linux 8.10 |
| CPU | AMD Ryzen 7 — 6 vCPUs |
| RAM | 6 GB |
| Disco OS | 100 GB LVM → `/u01` (Oracle Home + binários) |
| Disco Dados | 40 GB LVM → `/u02` (datafiles, redo logs, FRA) |
| Virtualização | VMware Workstation Pro |
| Rede | Definido em `.env` |

### Estrutura de diretórios Oracle
```
/u01/app/oracle/product/19.3.0/db_1       ← Oracle Home 19c inicial
/u01/app/oracle/product/19.3.0/dbhome_1   ← Oracle Home 19c patchado
/u01/app/oracle/product/23.0.0/dbhome_1   ← Oracle Home 23ai
/u01/app/oraInventory
/u02/oradata/                              ← Datafiles (OMF)
/u02/fra/                                 ← Fast Recovery Area
/install/                                 ← Gold Images e patches
```

---

## Como usar as skills

### 1. Configurar o ambiente
```bash
# Copiar template e editar
cp .env.template .env
# Editar .env com os valores reais (VM_IP, senhas, nomes de disco, etc.)
```

### 2. Configurar SSH (primeira vez)
```
/oracle-imersao setup-ssh
```

### 3. Executar atividade
```
/oracle-imersao 1        ← Atividade 1
/oracle-imersao 2        ← Atividade 2
...
/oracle-imersao 8        ← Atividade 8
```
Ou invocar a skill diretamente:
```
/01-prepare-ambiente
/02-download-patch
...
/08-patch-oracle-restart
```

---

## Pré-requisitos antes de iniciar

1. **`.env` preenchido** — copiar de `.env.template` e preencher TODOS os campos `CHANGE_ME`.
2. **Gold Image 19c** copiada para `/install` na VM antes da Atividade 1.
3. **Gold Image 23ai** copiada para `/install` na VM antes da Atividade 6.
4. **Patches** copiados para `/install/patches/` antes das atividades de patch.
5. **autoupgrade.jar** (versão mais recente) copiado para `/install/`.

---

## Segurança

- **Nunca commitar o arquivo `.env`** — ele contém senhas e está no `.gitignore`.
- Todas as skills mascaram senhas nos logs com `***`.
- SSH via chave privada (sem senha em linha de comando).
