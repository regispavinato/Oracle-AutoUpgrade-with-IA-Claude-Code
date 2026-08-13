---
description: Skill principal da imersão Oracle AutoUpgrade — roteador de atividades e setup inicial de SSH
---

# Oracle Imersão — Skill Principal

Você é o orquestrador da imersão Oracle AutoUpgrade de 2 dias. Sua função é:
1. Verificar pré-requisitos
2. Rotear para a skill da atividade solicitada
3. Configurar SSH na primeira execução

## Atividades disponíveis

| Arg | Skill | Descrição |
|---|---|---|
| `setup-ssh` | — | Configura autenticação SSH por chave para a VM |
| `0` | `00-criar-vm` | Criar VM no VirtualBox (100GB+40GB, 6GB RAM, 6 vCPU, bridge+IP estático) + instalar Oracle Linux 8.10 Server (sem GUI) via kickstart |
| `1` | `01-prepare-ambiente` | OS prep + LVM + Oracle 19c + DBCA + Listener + startup |
| `2` | `02-download-patch` | Download do último patch com AutoUpgrade |
| `3` | `03-criar-oracle-home` | Criar novo Oracle Home com AutoUpgrade |
| `4` | `04-aplicar-patch` | Aplicar patch com AutoUpgrade |
| `5` | `05-converter-multitenant` | Converter non-CDB → Multitenant |
| `6` | `06-migrar-23ai` | Migrar 19c → 23ai (23.26.1) |
| `7` | `07-patch-23262` | Patch 23.26.1 → 23.26.2 |
| `8` | `08-configurar-backup` | Configurar rotina de backup RMAN (FULL semanal + incremental via cron) |
| `9` | `09-verificacao-final` | Verificação final do ambiente + resumo LinkedIn |

---

## Instruções de execução

### Passo 1 — Carregar configuração

Use o Read tool para ler o arquivo `.env` em `D:\Imersao_Autoupgrade\.env`.
Extraia as seguintes variáveis e guarde-as em memória de trabalho:
- `VM_IP`, `VM_HOSTNAME`, `SSH_KEY`
- `ROOT_PWD`, `ORACLE_OS_PWD`, `ORACLE_PWD`
- `ORACLE_USER`, `ORACLE_GROUP`, `DBA_GROUP`
- `ORACLE_BASE`, `ORACLE_HOME_19C`, `ORACLE_HOME_19C_PATCHED`, `ORACLE_HOME_23AI`
- `ORACLE_SID`, `DISK_DEVICE`, `VG_NAME`, `LV_NAME`
- `INSTALL_DIR`, `GOLD_IMAGE_19C`, `GOLD_IMAGE_23AI`, `AUTOUPGRADE_JAR`
- `ORACLE_INVENTORY`, `ORADATA_DIR`, `FRA_DIR`, `AUTOUPGRADE_LOG_DIR`

Se o arquivo `.env` não existir, informe o usuário e instrua a copiar de `.env.template`.
Se algum campo ainda estiver como `CHANGE_ME`, avise quais campos faltam antes de continuar.

**Segurança**: nunca exibir no output os valores de `ROOT_PWD`, `ORACLE_OS_PWD` ou `ORACLE_PWD`. Substitua por `***` em qualquer log ou mensagem.

### Passo 2 — Verificar arg recebido

O argumento recebido (via `args`) pode ser:
- `setup-ssh` → executar o bloco **Setup SSH** abaixo
- `0` a `9` → invocar a skill correspondente usando o Skill tool

> **Nota:** `0` (`00-criar-vm`) roda **antes** de qualquer SSH existir — a VM ainda não
> existe nesse ponto. Rodar `setup-ssh` primeiro só para gerar o par de chaves local (passo
> 3.1) já é suficiente: a skill `00-criar-vm` injeta a chave pública direto no `authorized_keys`
> da VM via kickstart, dispensando a cópia manual do passo 3.2 quando esse fluxo é usado.

### Passo 3 — Se arg for setup-ssh

#### 3.1 — Gerar o par de chaves SSH (Claude executa via PowerShell tool)

**IMPORTANTE — usar PowerShell, não Bash.** No Windows, o Git Bash pode gerar a chave com passphrase acidental quando `-N ""` é usado. O PowerShell com Windows OpenSSH não tem esse problema.

```powershell
$keyPath = "$env:USERPROFILE\.ssh\oracle_imersao"

if (-not (Test-Path $keyPath)) {
    New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.ssh" | Out-Null
    ssh-keygen -t ed25519 -f $keyPath -N "" -C "oracle-imersao"
    Write-Output "Par de chaves gerado:"
    Write-Output "  Privada : $keyPath"
    Write-Output "  Publica : $keyPath.pub"
} else {
    Write-Output "Par de chaves ja existe em $keyPath"
}

# Verificar que a chave NAO tem passphrase
$check = ssh-keygen -y -P "" -f $keyPath 2>&1
if ($LASTEXITCODE -ne 0) {
    Write-Output "AVISO: chave tem passphrase. Removendo..."
    ssh-keygen -p -P '""' -N "" -f $keyPath
}

Write-Output "=== Chave publica ==="
Get-Content "$keyPath.pub"
```

#### 3.2 — Copiar a chave para a VM (via MobaXterm ou console da VM)

O `ssh-copy-id` é problemático no Windows (paths Git Bash vs Windows, input interativo). **Método mais confiável:**

Exibir as seguintes instruções para o usuário:

---
> **Ação necessária — copie a chave para a VM:**
>
> **Opção A — MobaXterm (recomendado):**
> 1. Abra o MobaXterm e clique em "Start local terminal"
> 2. Execute: `ssh root@IP_DA_VM` (usando a senha root)
> 3. Dentro da VM, cole os comandos abaixo:
>
> ```bash
> mkdir -p /root/.ssh && chmod 700 /root/.ssh
> echo "CONTEUDO_DA_CHAVE_PUBLICA" >> /root/.ssh/authorized_keys
> chmod 600 /root/.ssh/authorized_keys
> echo "Chave adicionada OK"
> ```
>
> **Opção B — Console VMware:**
> Acesse a VM diretamente pelo VMware e execute os mesmos comandos acima.
>
> O conteúdo da chave pública (`CONTEUDO_DA_CHAVE_PUBLICA`) foi exibido no passo 3.1.
---

Aguardar confirmação do usuário de que a chave foi adicionada.

#### 3.3 — Configurar ~/.ssh/config com keep-alive (Claude executa via Write tool)

Criar ou atualizar `C:\Users\<usuario>\.ssh\config` para evitar quedas SSH em operações longas.
**Não usar ControlMaster — não funciona no Windows com Git Bash.**

```
Host oracle-vm
    HostName <VM_IP>
    User oracle
    IdentityFile ~/.ssh/oracle_imersao
    ServerAliveInterval 30
    ServerAliveCountMax 3
    StrictHostKeyChecking no
    KexAlgorithms curve25519-sha256,ecdh-sha2-nistp256

Host oracle-vm-root
    HostName <VM_IP>
    User root
    IdentityFile ~/.ssh/oracle_imersao
    ServerAliveInterval 30
    ServerAliveCountMax 3
    StrictHostKeyChecking no
    KexAlgorithms curve25519-sha256,ecdh-sha2-nistp256
```

#### 3.4 — Verificar conexão sem senha (Claude executa via Bash tool)

```bash
SSH_KEY_PATH=$(grep "^SSH_KEY=" /d/Imersao_Autoupgrade/.env | cut -d= -f2 | sed 's|~|'"$HOME"'|')
VM_IP=$(grep "^VM_IP=" /d/Imersao_Autoupgrade/.env | cut -d= -f2)

ssh -i "${SSH_KEY_PATH}" \
    -o StrictHostKeyChecking=no \
    -o LogLevel=ERROR \
    -o BatchMode=yes \
    -o ConnectTimeout=10 \
    -o ServerAliveInterval=30 \
    -o ServerAliveCountMax=3 \
    "root@${VM_IP}" "echo 'SSH OK — conexao sem senha funcionando' && hostname && uptime"
```

- Se retornar `SSH OK`: confirmar ao usuário que o ambiente está pronto.
- Se retornar `Permission denied` ou `Host key verification failed`: orientar a verificar se o `ssh-copy-id` foi executado corretamente ou se o IP/hostname no `.env` está correto.

#### 3.5 — Configurar também acesso como oracle (opcional mas recomendado)

```bash
SSH_KEY_PATH=$(grep "^SSH_KEY=" /d/Imersao_Autoupgrade/.env | cut -d= -f2 | sed 's|~|'"$HOME"'|')
VM_IP=$(grep "^VM_IP=" /d/Imersao_Autoupgrade/.env | cut -d= -f2)
ORACLE_USER=$(grep "^ORACLE_USER=" /d/Imersao_Autoupgrade/.env | cut -d= -f2)

# Copiar a mesma chave pública para o usuário oracle via root (sem senha adicional)
ssh -i "${SSH_KEY_PATH}" -o StrictHostKeyChecking=no -o LogLevel=ERROR "root@${VM_IP}" "
  mkdir -p /home/${ORACLE_USER}/.ssh
  cat >> /home/${ORACLE_USER}/.ssh/authorized_keys << 'PUBKEY'
$(cat ${SSH_KEY_PATH}.pub)
PUBKEY
  chown -R ${ORACLE_USER}:oinstall /home/${ORACLE_USER}/.ssh
  chmod 700 /home/${ORACLE_USER}/.ssh
  chmod 600 /home/${ORACLE_USER}/.ssh/authorized_keys
  echo 'Chave copiada para usuario oracle'
"
```

Verificar acesso como oracle:
```bash
SSH_KEY_PATH=$(grep "^SSH_KEY=" /d/Imersao_Autoupgrade/.env | cut -d= -f2 | sed 's|~|'"$HOME"'|')
VM_IP=$(grep "^VM_IP=" /d/Imersao_Autoupgrade/.env | cut -d= -f2)
ORACLE_USER=$(grep "^ORACLE_USER=" /d/Imersao_Autoupgrade/.env | cut -d= -f2)

ssh -i "${SSH_KEY_PATH}" -o BatchMode=yes "${ORACLE_USER}@${VM_IP}" "echo 'Oracle SSH OK' && id"
```

### Passo 4 — Se arg for número de atividade (1-9)

Invoque a skill correspondente usando o Skill tool com o nome da skill da tabela acima.
Exemplo: se arg=1, invocar skill `01-prepare-ambiente`.

Antes de invocar, confirme com o usuário:
> "Vou executar a Atividade [N]: [descrição]. Confirmar?"

---

## Verificação de pré-requisitos (executar para qualquer atividade)

Execute via Bash para confirmar conectividade:
```bash
VM_IP=$(grep "^VM_IP=" /d/Imersao_Autoupgrade/.env | cut -d= -f2)
SSH_KEY=$(grep "^SSH_KEY=" /d/Imersao_Autoupgrade/.env | cut -d= -f2 | sed 's|~|'"$HOME"'|')

ssh -i "${SSH_KEY}" -o StrictHostKeyChecking=no -o LogLevel=ERROR -o BatchMode=yes -o ConnectTimeout=10 \
    "root@${VM_IP}" "hostname && uptime && df -h /u01 2>/dev/null || echo '/u01 nao montado'"
```

Se falhar, orientar o usuário a rodar `setup-ssh` primeiro.
