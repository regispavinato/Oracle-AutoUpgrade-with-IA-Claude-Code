# Oracle Imersão AutoUpgrade

Imersão prática sobre Oracle AutoUpgrade: provisionamento de VM, instalação, patching,
conversão para Multitenant e migração para Oracle 23ai — tudo via skills do Claude Code,
executadas contra uma VM real.

## Atividades e tempo médio de execução

Os tempos abaixo foram observados numa execução real de ponta a ponta (VM com 6 vCPUs/6GB
RAM compartilhados) e refletem o tempo de execução em si (instalador, DBCA, AutoUpgrade,
etc.) — não o tempo de leitura/acompanhamento. **Variam bastante conforme o hardware do
host e o quanto a VM está sob concorrência** (ver `CLAUDE.md` § Gestão de memória).
Atividades com AutoUpgrade `-mode deploy` são as mais sensíveis a isso.

| # | Skill | Atividade | Tempo médio |
|---|---|---|---|
| 0 | `00-criar-vm` | Criar VM no VirtualBox (100GB+40GB, 6GB RAM, 6 vCPU, bridge+IP estático) + instalar Oracle Linux 8.10 Server via kickstart | ~15–20 min |
| 1 | `01-prepare-ambiente` | OS prep + LVM `/u02` + instalação Oracle 19c non-CDB + DBCA + listener + startup | ~40–50 min |
| 2 | `02-download-patch` | Download do `autoupgrade.jar` mais recente + verificação do Oracle Home 19c | ~5–10 min |
| 3 | `03-criar-oracle-home` | Criar novo Oracle Home 19c a partir do Gold Image, pronto para patching | ~10–15 min |
| 4 | `04-aplicar-patch` | Atualizar OPatch + aplicar RU + mover o banco com AutoUpgrade (`deploy`) | ~25–35 min |
| 5 | `05-converter-multitenant` | Criar CDB vazio (DBCA) + converter non-CDB → PDB com AutoUpgrade (`noncdbtopdb`) + reorganizar datafiles (OMF) | ~50–60 min |
| 6 | `06-migrar-23ai` | Instalar Oracle 23ai + migrar 19c → 23.26.1 com AutoUpgrade (upgrade completo de dicionário) | ~75–90 min |
| 7 | `07-patch-23262` | Instalar novo Oracle Home + aplicar patch 23.26.1 → 23.26.2 com AutoUpgrade | ~30–40 min |
| 8 | `08-configurar-backup` | Configurar RMAN (retenção, canal, controlfile autobackup) + script + cron (FULL semanal / incremental diário) | ~15 min (1ª execução, nível 0 completo — incrementais diárias subsequentes são bem mais rápidas) |

**Total estimado, ponta a ponta**: ~4h30–6h de execução real (sem contar as pausas entre
atividades). As atividades 5 e 6 concentram a maior parte do tempo — envolvem DBCA e
upgrade completo de dicionário, respectivamente, e são as mais dependentes do hardware.

---

## Ambiente

| Componente | Detalhe |
|---|---|
| SO da VM | Oracle Linux 8.10 Server (minimal, sem GUI) |
| CPU da VM | 6 vCPUs |
| RAM da VM | 6 GB |
| Disco OS | 100 GB → `/` (LVM, criado pelo instalador) |
| Disco Dados | 40 GB LVM → `/u02` (datafiles, redo logs, FRA) |
| Hypervisor | VirtualBox (`VBoxManage`, rodando no host) — VM provisionada pela skill `00-criar-vm` |
| Rede | Bridge (adaptador do host) + IP estático via kickstart — definido em `.env` |

### Estrutura de diretórios Oracle (estado final, após a Atividade 7)
```
/u01/app/oracle/product/23.0.0/dbhome_2   ← Oracle Home 23.26.2 (ativo)
/u01/app/oraInventory
/u02/oradata/CDBORCL/                      ← Datafiles do CDB e da PDB ORCLPDB (OMF)
/u02/fra/                                 ← Fast Recovery Area
/install/                                 ← Gold Images e patches
/backup/                                  ← Backups RMAN (Atividade 8)
```
Os Oracle Homes intermediários (`19.3.0/db_1`, `19.3.0/dbhome_1`, `23.0.0/dbhome_1`) são
removidos ao longo do processo, conforme cada atividade migra o banco para o home seguinte
— ver `CLAUDE.md` § Transição de ORACLE_SID para o mapeamento de SID por atividade.

---

## Como usar as skills

### 1. Configurar o ambiente
```bash
# Copiar template e editar
cp ".env - Copia.template" .env
# Editar .env com os valores reais (VM_IP, senhas, ISO_PATH_OL8, etc.)
```

### 2. Configurar SSH (primeira vez)
```
/oracle-imersao setup-ssh
```

### 3. Executar atividade
```
/oracle-imersao 0        ← Atividade 0 (criar a VM — só precisa rodar uma vez)
/oracle-imersao 1        ← Atividade 1
/oracle-imersao 2        ← Atividade 2
...
/oracle-imersao 8        ← Atividade 8
```
Ou invocar a skill diretamente:
```
/00-criar-vm
/01-prepare-ambiente
/02-download-patch
/03-criar-oracle-home
/04-aplicar-patch
/05-converter-multitenant
/06-migrar-23ai
/07-patch-23262
/08-configurar-backup
```

---

## Pré-requisitos antes de iniciar

1. **`.env` preenchido** — copiar do template e preencher TODOS os campos `CHANGE_ME`,
   incluindo o bloco `VM VirtualBox` (`ISO_PATH_OL8`, `VM_GATEWAY`, `VM_BRIDGE_ADAPTER`, etc.).
2. **VirtualBox instalado** no host, com um adaptador de rede em modo bridge disponível.
3. **ISO do Oracle Linux 8.10** baixada e apontada em `ISO_PATH_OL8` (Atividade 0).
4. **Gold Image 19c** copiada para `/install` na VM antes da Atividade 1.
5. **Gold Image 23ai (23.26.1)** copiada para `/install` na VM antes da Atividade 6.
6. **Gold Image completa 23.26.2** (`GOLD_IMAGE_23AI_PATCHED`) copiada para `/install` antes da Atividade 7.
7. **OPatch + RU do 19c** copiados para `/install` antes da Atividade 3/4.
8. **autoupgrade.jar** — a Atividade 1 já copia a versão inicial; a Atividade 2 atualiza para a mais recente.

---

## Segurança

- **Nunca commitar o arquivo `.env`** — ele contém senhas e está no `.gitignore`.
- Todas as skills mascaram senhas nos logs com `***`.
- SSH via chave privada (sem senha em linha de comando).
- O `ks.cfg` gerado pela Atividade 0 contém a senha de root em texto plano (exigência do
  formato kickstart) — fica fora do repositório do projeto, nunca é commitado nem exibido.
