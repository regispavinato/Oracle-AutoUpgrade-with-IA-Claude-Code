---
description: Atividade 0 — Criar VM no VirtualBox (100GB+40GB, 6GB RAM, 6 vCPU, rede bridge com IP estático) e instalar Oracle Linux 8.10 Server (sem GUI) via kickstart, antes da Atividade 1
---

# Atividade 0 — Criar VM no VirtualBox + Instalar Oracle Linux 8.10 (Server, sem GUI)

Esta atividade roda **no host Windows** (não via SSH — a VM ainda não existe). Todos os
comandos `VBoxManage` usam o **PowerShell tool**, nunca o Bash tool (mesma regra do
`ssh-keygen` na skill principal `oracle-imersao`).

## Pré-requisitos
- VirtualBox instalado no host (`VBoxManage --version`).
- `.env` preenchido, incluindo o bloco `--- VM VirtualBox (skill 00-criar-vm) ---`.
- `ISO_PATH_OL8` apontando para um `.iso` do Oracle Linux 8.10 já baixado no host.
- (Opcional, recomendado) Chave SSH local já gerada pela skill `oracle-imersao` (seção
  "Setup SSH", passo 3.1 — só a geração, **não** precisa copiar a chave manualmente):
  `%USERPROFILE%\.ssh\oracle_imersao.pub`. Se existir, esta skill injeta a chave pública
  direto no `authorized_keys` do root via kickstart, dispensando a cópia manual descrita
  na seção 3.2 da `oracle-imersao`.

## Carregar variáveis

Leia `.env` com o Read tool e extraia: `VM_HOSTNAME`, `VM_IP`, `VM_NETMASK`, `VM_GATEWAY`,
`VM_DNS1`, `VM_DNS2`, `VM_BRIDGE_ADAPTER`, `VM_TIMEZONE`, `VM_RAM_MB`, `VM_CPUS`,
`VM_DISK1_GB`, `VM_DISK2_GB`, `VMS_BASE_DIR`, `ISO_PATH_OL8`, `ROOT_PWD`, `DISK_DEVICE`.

Se `ISO_PATH_OL8` estiver `CHANGE_ME` ou o arquivo não existir, **parar** e pedir para o
usuário preencher o caminho do `.iso` no `.env` antes de continuar.

**Segurança**: `ROOT_PWD` vai para dentro do `ks.cfg` em texto plano (exigência do
kickstart). Nunca exibir o conteúdo do `ks.cfg` no output — se precisar mostrar o arquivo
para o usuário verificar, mostrar com a linha `rootpw` mascarada (`rootpw --plaintext ***`).
O `ks.cfg` fica em `${VMS_BASE_DIR}\${VM_HOSTNAME}\ks.cfg`, fora do repositório do projeto
— nunca copiar esse arquivo para dentro do projeto nem versioná-lo.

Definir internamente (PowerShell):
```powershell
$VBM = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
```

---

## FASE 1 — Verificações antes de criar

### 1.1 — VirtualBox instalado

```powershell
& $VBM --version
```

### 1.2 — VM com esse nome já existe?

```powershell
& $VBM list vms | Select-String -Quiet '"${VM_HOSTNAME}"'
```
Se já existir, avisar o usuário e perguntar se deve apagar (`VBoxManage unregistervm
"${VM_HOSTNAME}" --delete`) antes de prosseguir — nunca apagar sem confirmação.

### 1.3 — Adaptador bridge existe e está ativo

```powershell
& $VBM list bridgedifs | Select-String -Pattern "Name:|IPAddress:|Status:"
```
Confirmar que `${VM_BRIDGE_ADAPTER}` aparece com `Status: Up`.

### 1.4 — ISO existe

```powershell
Test-Path "${ISO_PATH_OL8}"
```

---

## FASE 2 — Criar a VM (shell) e configurar hardware

### 2.1 — Criar a VM e registrar

```powershell
& $VBM createvm --name "${VM_HOSTNAME}" --platform-architecture=x86 --ostype "Oracle8_64" --basefolder "${VMS_BASE_DIR}" --register
```

### 2.2 — CPU, memória, firmware, boot order

```powershell
& $VBM modifyvm "${VM_HOSTNAME}" `
  --memory ${VM_RAM_MB} --cpus ${VM_CPUS} --ioapic on `
  --firmware bios `
  --boot1 dvd --boot2 disk --boot3 none --boot4 none `
  --graphicscontroller vmsvga --vram 16 `
  --audio-enabled off `
  --usb-ehci off --usb-xhci off
```
VM é um servidor headless — sem áudio, sem USB 2.0/3.0, firmware BIOS (compatível com o
particionamento MBR do kickstart, evita complicações de Secure Boot/EFI).

### 2.3 — Rede bridge com adaptador correto

```powershell
& $VBM modifyvm "${VM_HOSTNAME}" --nic1=bridged --bridge-adapter1="${VM_BRIDGE_ADAPTER}" --nic-type1=82540EM --cable-connected1=on
```
O IP estático em si é configurado dentro do guest pelo kickstart (FASE 4), não pelo
VirtualBox — a placa bridge só coloca a VM na mesma rede física do host.

---

## FASE 3 — Discos

### 3.1 — Controller SATA

```powershell
& $VBM storagectl "${VM_HOSTNAME}" --name "SATA Controller" --add sata --controller IntelAhci --portcount 3 --bootable on
```

### 3.2 — Disco 1 — SO (${VM_DISK1_GB}GB, dynamic VDI) — porta 0

Dynamically-allocated: só ocupa espaço em disco no host conforme uso real dentro da VM
(o tamanho de ${VM_DISK1_GB}GB é o limite virtual, não a alocação imediata).

```powershell
$disk1 = "${VMS_BASE_DIR}\${VM_HOSTNAME}\${VM_HOSTNAME}_disk1_os.vdi"
& $VBM createmedium disk --filename "$disk1" --size ([int]${VM_DISK1_GB} * 1024) --format VDI --variant Standard
& $VBM storageattach "${VM_HOSTNAME}" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium "$disk1"
```

### 3.3 — Disco 2 — dados (${VM_DISK2_GB}GB, dynamic VDI) — porta 1

**Não particionar nem formatar este disco aqui.** Ele deve chegar cru (raw) na
Atividade 1, que faz `pvcreate`/`vgcreate`/`lvcreate` sobre `${DISK_DEVICE}` (Fase 2 da
`01-prepare-ambiente`). Este disco vira `/dev/sdb` dentro do guest.

```powershell
$disk2 = "${VMS_BASE_DIR}\${VM_HOSTNAME}\${VM_HOSTNAME}_disk2_data.vdi"
& $VBM createmedium disk --filename "$disk2" --size ([int]${VM_DISK2_GB} * 1024) --format VDI --variant Standard
& $VBM storageattach "${VM_HOSTNAME}" --storagectl "SATA Controller" --port 1 --device 0 --type hdd --medium "$disk2"
```

### 3.4 — Controller IDE + ISO de instalação

```powershell
& $VBM storagectl "${VM_HOSTNAME}" --name "IDE Controller" --add ide
& $VBM storageattach "${VM_HOSTNAME}" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium "${ISO_PATH_OL8}"
```

---

## FASE 4 — Gerar o kickstart (`ks.cfg`)

### 4.1 — Ler a chave pública local (se existir)

```powershell
$pubKeyPath = "$env:USERPROFILE\.ssh\oracle_imersao.pub"
$pubKey = if (Test-Path $pubKeyPath) { Get-Content $pubKeyPath -Raw } else { $null }
```

### 4.2 — Escrever o `ks.cfg`

Use o Write tool para criar `${VMS_BASE_DIR}\${VM_HOSTNAME}\ks.cfg` com o conteúdo abaixo,
substituindo `${...}` pelos valores do `.env`. **Não** imprimir este arquivo no output com
a senha em texto plano — se for mostrar para o usuário, mascare a linha `rootpw`.

```
lang en_US.UTF-8
keyboard us
timezone ${VM_TIMEZONE} --utc
rootpw --plaintext ${ROOT_PWD}
network --bootproto=static --device=link --activate --ip=${VM_IP} --netmask=${VM_NETMASK} --gateway=${VM_GATEWAY} --nameserver=${VM_DNS1},${VM_DNS2} --hostname=${VM_HOSTNAME}
selinux --disabled
firewall --disabled
ignoredisk --only-use=sda
clearpart --all --initlabel --drives=sda
part /boot --fstype=xfs --size=1024
part pv.sda1 --size=1 --grow
volgroup vg_sys pv.sda1
logvol swap --vgname=vg_sys --name=lv_swap --size=${VM_RAM_MB}
logvol / --vgname=vg_sys --name=lv_root --size=1 --grow --fstype=xfs
bootloader --location=mbr --boot-drive=sda
skipx
reboot

%packages
@^minimal-environment
%end

%post --log=/root/ks-post.log
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
systemctl enable sshd
mkdir -p /root/.ssh
chmod 700 /root/.ssh
%end
```

Se `$pubKey` não for nulo, adicionar estas linhas **dentro** do bloco `%post`, antes do
`%end`, com o conteúdo real da chave:
```
cat >> /root/.ssh/authorized_keys << 'PUBKEY_EOF'
<conteúdo de $pubKey aqui>
PUBKEY_EOF
chmod 600 /root/.ssh/authorized_keys
```
Sem isso, a cópia da chave fica pendente para a seção 3.2 (manual, via MobaXterm) da skill
`oracle-imersao`.

**Nota (2026-08-14):** o particionamento é manual (`part`/`logvol`), não `autopart
--type=lvm`, de propósito — o `autopart` padrão do Anaconda cria um LV **`/home` separado**
sempre que o disco é grande o bastante (é o caso aqui, 100GB), reservando ~30GB que essa VM
nunca usa (é um servidor Oracle, não tem usuários locais de verdade). Com `part`/`logvol`
explícitos, só existem `/boot` (1GB, fora do LVM), swap (= `${VM_RAM_MB}` MB) e `/` (resto
do disco) — `/home` continua existindo, só que como diretório comum dentro de `/`, não como
mount/LV separado. Da mesma forma, `/u01` (Fase 3 da `01-prepare-ambiente`) também é só um
diretório dentro de `/`, não um mount separado. `ignoredisk --only-use=sda` +
`clearpart --drives=sda` garantem que `sdb` (o disco de ${VM_DISK2_GB}GB) nunca é tocado
pelo instalador.

**Ainda não testado ponta-a-ponta** (2026-08-14): a VM usada para validar esta skill até
aqui (`srv01.localdomain`) foi criada com o `autopart --type=lvm` antigo; a segunda VM de
teste (`srv02.localdomain`) foi apagada antes de chegar a rodar um kickstart com este
`part`/`logvol` novo. Sintaxe conferida contra a documentação do pykickstart, mas a
primeira execução real deste bloco deve ser acompanhada com atenção — em especial se
`logvol swap --size=${VM_RAM_MB}` aceita a variável numérica sem problema, e se o
`bootloader --boot-drive=sda` ainda funciona sem o `/boot` vindo do `autopart`.

`@^minimal-environment` é o grupo de pacotes "Minimal Install" do Anaconda — sem
GNOME/X11/nenhum pacote gráfico, mesmo padrão de um servidor.

---

## FASE 5 — Rodar a instalação unattended

Montar os argumentos como array e usar splatting — continuação de linha com backtick
`` ` `` já causou "Empty user password is not allowed" por embaralhar argumentos silenciosamente:

```powershell
$argsList = @(
  "unattended", "install", "${VM_HOSTNAME}",
  "--iso=${ISO_PATH_OL8}",
  "--script-template=${VMS_BASE_DIR}\${VM_HOSTNAME}\ks.cfg",
  "--hostname=${VM_HOSTNAME}",
  "--user=root",
  "--user-password=placeholder-unused",
  "--no-install-additions",
  "--no-install-txs",
  "--package-selection-adjustment=minimal",
  "--start-vm=headless"
)
& $VBM @argsList
```
`--user`/`--user-password` são obrigatórios mesmo usando `--script-template` custom (que já
define `rootpw` sozinho) — sem eles o VBoxManage recusa com "Empty user password is not
allowed" antes de sequer processar o template. O valor de `--user-password` aqui não é
usado de fato (o `ks.cfg` manda).

Este comando configura a VM para instalação (anexa o `ks.cfg` gerado via mídia auxiliar) e
liga a VM sem abrir janela (`headless`). O comando **retorna assim que a VM é ligada** —
não espera a instalação terminar. Isso é feito na Fase 6.

---

## FASE 6 — Aguardar a instalação terminar

A instalação (kickstart + `reboot` no final) costuma levar de 10 a 20 minutos. Não há
callback nativo sem TXS instalado — o sinal mais confiável é a porta 22 responder.

```powershell
$deadline = (Get-Date).AddMinutes(30)
$up = $false
while ((Get-Date) -lt $deadline) {
    $result = Test-NetConnection -ComputerName "${VM_IP}" -Port 22 -WarningAction SilentlyContinue
    if ($result.TcpTestSucceeded) { $up = $true; break }
    Start-Sleep -Seconds 30
}
if ($up) { Write-Output "SSH disponivel em ${VM_IP}:22" } else { Write-Output "TIMEOUT aguardando SSH" }
```
Se der timeout, verificar console da VM (`VBoxManage startvm "${VM_HOSTNAME}" --type gui` — só
para diagnóstico visual; religar em headless depois) antes de insistir.

---

## FASE 7 — Verificação

### 7.1 — Login (se a chave pública foi injetada na Fase 4)

```bash
ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o LogLevel=ERROR root@${VM_IP} "hostname -f; cat /etc/os-release | grep -E '^(NAME|VERSION)='"
```
Se a chave **não** foi injetada, seguir a seção 3.2 da skill `oracle-imersao` (cópia manual
via senha) antes de continuar.

### 7.2 — Confirmar IP, discos e ausência de GUI

```bash
ssh -i ${SSH_KEY} root@${VM_IP} "
echo '=== IP ==='
ip -4 -o addr show | grep -v '127.0.0.1'
echo '=== Discos ==='
lsblk -d -o NAME,SIZE,TYPE
echo '=== sdb sem particao/mount (esperado para a Atividade 1) ==='
lsblk /dev/sdb
echo '=== default target (esperado: multi-user.target, sem GUI) ==='
systemctl get-default
echo '=== pacotes graficos (esperado: vazio) ==='
rpm -qa | grep -i gnome
"
```

**Atividade 0 concluída com sucesso quando:**
- `ip -4 -o addr` mostra `${VM_IP}/24` na interface bridged.
- `lsblk` mostra `sda` (~${VM_DISK1_GB}G, com partições/LVM) e `sdb` (~${VM_DISK2_GB}G,
  **sem** partições nem punto de montagem).
- `systemctl get-default` retorna `multi-user.target`.
- `rpm -qa | grep -i gnome` não retorna nada.
- SSH como root responde (por chave ou por senha, conforme Fase 4).

## FASE 8 — Ejetar o ISO e reiniciar limpo

O controller IDE não suporta hot-unplug — `storageattach ... --medium none` com a VM ligada
falha com "does not support hot-plugging". Por isso este passo é: desligar limpo → ejetar →
ligar de novo → confirmar SSH. Não pular esta fase — deixar o ISO de 13GB permanentemente
anexado (às vezes montado a partir de uma pasta sincronizada tipo Google Drive/OneDrive)
deixa a VM refém da disponibilidade desse arquivo/unidade toda vez que ligar.

### 8.1 — Desligamento limpo via SSH

```bash
ssh -i ${SSH_KEY} root@${VM_IP} "shutdown -h now" 2>/dev/null || true
```

### 8.2 — Aguardar VMState=poweroff

```powershell
$deadline = (Get-Date).AddMinutes(3)
$off = $false
while ((Get-Date) -lt $deadline) {
    $state = (& $VBM showvminfo "${VM_HOSTNAME}" --machinereadable | Select-String '^VMState=').ToString()
    if ($state -match 'poweroff') { $off = $true; break }
    Start-Sleep -Seconds 5
}
if (-not $off) { & $VBM controlvm "${VM_HOSTNAME}" poweroff; Start-Sleep -Seconds 5 }
```
Se o `shutdown -h now` não desligar dentro de 3 minutos, o `controlvm poweroff` força
(equivalente a desligar no botão — aceitável aqui pois a instalação já terminou e não há
banco de dados rodando ainda).

### 8.3 — Ejetar o ISO

```powershell
& $VBM storageattach "${VM_HOSTNAME}" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium none
```

### 8.4 — Ligar de novo (headless) e confirmar SSH

```powershell
& $VBM startvm "${VM_HOSTNAME}" --type headless
```
Se retornar `VBOX_E_INVALID_OBJECT_STATE` / "already locked by a session (or being locked
or unlocked)" logo após o `storageattach` do passo anterior, é a sessão do
`storageattach`/`poweroff` ainda liberando — aguardar ~5s e tentar `startvm` de novo, sem
precisar refazer o eject.

Repetir o polling de porta 22 da Fase 6 (normalmente volta em menos de 1 minuto, já que não
há kickstart para rodar de novo — é só um boot normal). Depois, repetir 7.1/7.2 para
confirmar que nada mudou (IP, discos, `multi-user.target`).

### 8.5 — Handoff para a Atividade 1

Informar o usuário: VM criada e OS instalado. Antes de rodar `/oracle-imersao 1`, copiar a
Gold Image 19c (`${GOLD_IMAGE_19C}`) para `${INSTALL_DIR}` dentro da VM.

---

## Notas / Troubleshooting

- **`sda`/`sdb` podem trocar de ordem entre boots** (confirmado 2026-08-14, segunda VM
  criada com esta skill): mesmo com o disco 1 (SO, 100GB) corretamente na porta 0 do SATA
  Controller e o disco 2 (dados, ${VM_DISK2_GB}GB) na porta 1 — confirmado via
  `VBoxManage showvminfo --machinereadable`, configuração do hypervisor correta — o kernel
  Linux pode nomear os discos na ordem inversa num boot específico (não-determinístico,
  ligado à ordem em que o AHCI virtualizado responde ao probe, mais sensível logo após
  mudar algo no barramento como ejetar o ISO do IDE). O disco em si não muda, só o nome
  que o kernel dá a ele naquele boot. Um reboot subsequente pode voltar ao normal sozinho
  (foi o que aconteceu), mas **nunca assumir a letra fixa** — sempre conferir dinamicamente
  qual disco está livre antes de qualquer operação destrutiva (a skill `01-prepare-ambiente`
  já faz essa checagem por uso, não por nome, na Fase 2.1 — é a proteção real contra isso).
  Se `DISK_DEVICE` no `.env` não bater com o que a checagem dinâmica mostrar, atualizar o
  `.env` antes de prosseguir, nunca seguir com o valor desatualizado.
- **`--platform-architecture=x86` é obrigatório** em `createvm` a partir do VirtualBox 7.x
  — sem essa flag o comando falha com parâmetro faltando.
- **Sintaxe de rede no `modifyvm` mudou** nas versões recentes do VirtualBox: é
  `--nic1=bridged` e `--bridge-adapter1="..."` (com `=`), não mais `--nic1 bridged
  --bridgeadapter1 "..."` (com espaço) das versões antigas — confirme com `VBoxManage
  modifyvm` (sem argumentos) se o comportamento não bater com o esperado, a sintaxe pode
  mudar entre versões.
- **`--size` do `createmedium` é em megabytes**, não gigabytes — sempre multiplicar
  `VM_DISK{1,2}_GB * 1024`.
- **`VBoxManage unattended install` exige `--user-password` não-vazio mesmo com
  `--script-template` custom** (que já define `rootpw` sozinho) — sem passar
  `--user=root --user-password=<qualquer-valor>` explicitamente, falha com "Empty user
  password is not allowed" antes mesmo de tentar processar o template. O valor passado
  aqui não é usado (o `ks.cfg` customizado manda), mas o flag precisa existir.
- **Em PowerShell, montar o comando `VBoxManage unattended install` como array e usar
  splatting (`& $VBM @argsList`)**, não continuação de linha com backtick — um backtick
  seguido de espaço invisível quebra a continuação silenciosamente e embaralha os
  argumentos (foi a causa raiz do erro de senha vazia acima na primeira tentativa).
- **O `unattended install` reordena `--boot1` para `disk` sozinho** ao preparar a VM —
  não é necessário (nem gera efeito, pois é sobrescrito) fixar a ordem de boot manualmente
  depois da instalação.
- **Ejetar o ISO do controller IDE exige a VM desligada** — `storageattach ... --medium
  none` com a VM rodando falha com "does not support hot-plugging" (IDE não suporta
  hot-unplug). Não bloqueia nada (ver 7.3), mas não tente com a VM ligada.
- **Discos dynamic (thin-provisioned)**: o espaço livre no host pode não comportar
  ${VM_DISK1_GB}+${VM_DISK2_GB}GB se ambos os discos encherem ao longo da imersão
  (instalação Oracle 19c+23ai, patches, RMAN). Monitorar espaço livre em
  `${VMS_BASE_DIR}` com `Get-PSDrive` periodicamente nas atividades seguintes.
- **Firmware BIOS, não EFI**: o kickstart usa `bootloader --location=mbr`, que assume BIOS
  legado. Se a VM for criada com `--firmware efi`, o boot falha após a instalação.
- **Validado ponta-a-ponta em 2026-08-13**: `createvm` → discos → `ks.cfg` (com injeção da
  chave pública) → `unattended install` → instalação real com o ISO de 13GB (montado a
  partir de uma pasta sincronizada do Google Drive, não disco local) → boot → login SSH por
  chave funcionando de primeira → todos os critérios da FASE 7 conferidos (IP, `sda` 100G,
  `sdb` 40G intocado, hostname, `multi-user.target`, zero pacotes GNOME, `sshd_config`
  correto). Duração exata da instalação não foi cronometrada nesta rodada.
