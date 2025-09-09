# Tech speed Up - azure-automation-account
Laboratório para praticar o Azure automation account no Azure


# Laboratório 1 - Azure Automation Account

## Ligar e desligar uma VM no horário comercial

**Objetivo:**  
Manter uma VM ligada das 07:30 às 18:30 e desligada no restante do tempo, de forma automática e confiável.

---

## Sumário
- [Pré-requisitos](#pré-requisitos)
- [Arquitetura](#arquitetura)
- [Passo a passo](#passo-a-passo)
  - [1. Criar Resource Group](#1-criar-um-resource-group)
  - [2. Criar Automation Account](#2-criar-a-automation-account)
  - [3. Conceder RBAC mínimo](#3-conceder-rbac-mínimo-à-managed-identity)
  - [4. Criar Automation Variables](#4-criar-automation-variables)
  - [5. Criar Runbooks](#5-criar-runbooks)
    - [Start-VM.ps1](#a-start-vmps1)
    - [Stop-VM.ps1](#b-stop-vmps1)
  - [6. Criar Schedules](#6-criar-schedules)
  - [7. Vincular Schedules aos Runbooks](#7-vincular-schedules-aos-runbooks)
  - [8. Testes e validação](#8-testes-e-validação)
  - [9. Limpeza do ambiente](#9-limpeza-do-ambiente)
- [Conclusão](#conclusão)

---

## Pré-requisitos
1. Conta no Azure com permissão para criar recursos (RG, Automation Account, RBAC).  
2. Uma VM no Azure (use o tier mais barato para testes).  
3. Recomenda-se criar tudo em um Resource Group de laboratório para facilitar a limpeza depois.  

---

## Arquitetura
O que iremos usar para executar a tarefa:

1. Automation Account com Managed Identity (System Assigned).  
2. RBAC: a identidade gerenciada recebe **Virtual Machine Contributor** no escopo da VM.  
3. Runbooks PowerShell (7.2): `Start-VM.ps1` e `Stop-VM.ps1`.  
4. Schedules:  
   - **07:30** → executa `Start-VM`.  
   - **18:30** → executa `Stop-VM`.  
   - Fuso horário: **São Paulo (UTC-03:00)**.  

---

## Passo a passo

### 1. Criar um Resource Group
Pelo portal ou CLI.

---

### 2. Criar a Automation Account
Portal do Azure → Create a resource → Automation → Automation Account

- Nome: `aa-lab-startstop`  
- Resource Group: seu RG  
- Region: a mesma da VM  
- Identity: habilite **System assigned**  
- Networking: mantenha **public access**  
- Tags: `env:lab` (opcional, mas recomendado)  

---

### 3. Conceder RBAC mínimo à Managed Identity
Portal → VM alvo → **Access control (IAM)** → **Add role assignment**

- Role: **Virtual Machine Contributor**  
- Assign access to: **User, group, or service principal**  
- Select members: escolha a **Managed Identity** do Automation Account  

---

### 4. Criar Automation Variables
Na **Automation Account → Shared Resources → Variables → Add a variable**:

- `SubscriptionId` (string) → valor da sua subscription  
- `ResourceGroupName` (string) → nome do RG da VM  
- `VmName` (string) → nome da VM  

---

## 5. Criar Runbooks

### A) Start-VM.ps1
Portal → Automation Account → **Process Automation → Runbooks → Create a runbook**

- Nome: `start-vm`  
- Tipo: **PowerShell**  
- Runtime: **PowerShell 7.2**  
- Tags: `env:lab`  

**O que este script faz:**
1. Lê três variáveis do Automation (`SubscriptionId`, `ResourceGroupName`, `VmName`) para saber exatamente **qual VM** operar.
2. Valida que essas variáveis existem e não estão vazias.
3. Faz login com a **Managed Identity** da sua Automation Account (RBAC mínimo na VM).
4. Pergunta ao Azure qual é o **estado atual** da VM (ligada/desligada) usando a API (`instanceView`).
5. Se a VM **já estiver ligada**, ele não faz nada (idempotente).
6. Se estiver desligada, chama a **API de Start** e finaliza.

```powershell
param([object]$WebhookData)
$ErrorActionPreference = 'Stop'

function Assert-Var($name, $value) {
  if ([string]::IsNullOrWhiteSpace([string]$value)) {
    throw "A variável de Automation '$name' está vazia ou ausente."
  }
}

# Variáveis
$SubscriptionIdRaw    = Get-AutomationVariable -Name 'SubscriptionId'
$ResourceGroupNameRaw = Get-AutomationVariable -Name 'ResourceGroupName'
$VmNameRaw            = Get-AutomationVariable -Name 'VmName'

Assert-Var 'SubscriptionId'    $SubscriptionIdRaw
Assert-Var 'ResourceGroupName' $ResourceGroupNameRaw
Assert-Var 'VmName'            $VmNameRaw

$SubscriptionId    = ($SubscriptionIdRaw -replace '[\r\n]', '').Trim()
$ResourceGroupName = ($ResourceGroupNameRaw -replace '[\r\n]', '').Trim()
$VmName            = ($VmNameRaw -replace '[\r\n]', '').Trim()

try { [void][Guid]::Parse($SubscriptionId) } catch { throw "SubscriptionId inválido: '$SubscriptionIdRaw'" }

Write-Output "Start-VM | Sub: $SubscriptionId | RG: $ResourceGroupName | VM: $VmName"

# Login com a Managed Identity vinculada à Automation Account
Connect-AzAccount -Identity | Out-Null

$vmId = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Compute/virtualMachines/$VmName"
$api = "2023-07-01"

# Lê o estado atual da VM
$ivResp = Invoke-AzRest -Path "$vmId/instanceView?api-version=$api" -Method GET
if ($ivResp.StatusCode -lt 200 -or $ivResp.StatusCode -ge 300) {
  throw "Falha ao obter instanceView. HTTP $($ivResp.StatusCode) $($ivResp.Content)"
}
$iv = $ivResp.Content | ConvertFrom-Json
$power = ($iv.statuses | Where-Object { $_.code -like 'PowerState/*' }).displayStatus
Write-Output "PowerState atual: $power"

if ($power -eq 'VM running') {
  Write-Output "Já está ligada. Nada a fazer."
  return
}

# Aciona start
$resp = Invoke-AzRest -Path "$vmId/start?api-version=$api" -Method POST
if ($resp.StatusCode -lt 200 -or $resp.StatusCode -ge 300) {
  throw "Falha ao iniciar VM. HTTP $($resp.StatusCode) $($resp.Content)"
}
Write-Output "Start acionado (202/200)."
```

**Por que usar `Invoke-AzRest`?**  
Funciona de forma direta com a API do Azure e evita dependência de muitos módulos. É rápido, previsível e fácil de debugar (você vê o **StatusCode** e o **Content**).

**Dicas rápidas de troubleshooting:**
- Se aparecer erro de permissão, verifique o **RBAC** da Managed Identity na VM (Virtual Machine Contributor).  
- Confirme se as variáveis estão corretas (sem espaço extra/linha quebrada).  
- O nome da VM é **case-insensitive**, mas o **RG** e o **SubscriptionId** precisam estar 100% corretos.  

---

### B) Stop-VM.ps1
Portal → Automation Account → **Process Automation → Runbooks → Create a runbook**

- Nome: `stop-vm`  
- Tipo: **PowerShell**  
- Runtime: **PowerShell 7.2**  
- Tags: `env:lab`  

**O que este script faz:**
1. Lê as mesmas três variáveis para identificar a VM.  
2. Faz login com a Managed Identity.  
3. Checa o estado atual (instanceView).  
4. Se a VM **já estiver desligada/dealocada**, ele não faz nada.  
5. Caso contrário, chama a **API de Deallocate** (economiza computação; o disco continua sendo cobrado).  

```powershell
param([object]$WebhookData)
$ErrorActionPreference = 'Stop'

function Assert-Var($name, $value) {
  if ([string]::IsNullOrWhiteSpace([string]$value)) {
    throw "A variável de Automation '$name' está vazia ou ausente."
  }
}

# Variáveis
$SubscriptionIdRaw    = Get-AutomationVariable -Name 'SubscriptionId'
$ResourceGroupNameRaw = Get-AutomationVariable -Name 'ResourceGroupName'
$VmNameRaw            = Get-AutomationVariable -Name 'VmName'

Assert-Var 'SubscriptionId'    $SubscriptionIdRaw
Assert-Var 'ResourceGroupName' $ResourceGroupNameRaw
Assert-Var 'VmName'            $VmNameRaw

$SubscriptionId    = ($SubscriptionIdRaw -replace '[\r\n]', '').Trim()
$ResourceGroupName = ($ResourceGroupNameRaw -replace '[\r\n]', '').Trim()
$VmName            = ($VmNameRaw -replace '[\r\n]', '').Trim()

Write-Output "Stop-VM | Sub: $SubscriptionId | RG: $ResourceGroupName | VM: $VmName"

# Login com a Managed Identity vinculada à Automation Account
Connect-AzAccount -Identity | Out-Null

$vmId = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Compute/virtualMachines/$VmName"
$api = "2023-07-01"

# Lê o estado atual da VM
$ivResp = Invoke-AzRest -Path "$vmId/instanceView?api-version=$api" -Method GET
if ($ivResp.StatusCode -lt 200 -or $ivResp.StatusCode -ge 300) {
  throw "Falha ao obter instanceView. HTTP $($ivResp.StatusCode) $($ivResp.Content)"
}
$iv = $ivResp.Content | ConvertFrom-Json
$power = ($iv.statuses | Where-Object { $_.code -like 'PowerState/*' }).displayStatus
Write-Output "PowerState atual: $power"

if ($power -eq 'VM deallocated' -or $power -eq 'VM stopped') {
  Write-Output "Já está desligada/dealocada. Nada a fazer."
  return
}

# Aciona deallocate
$resp = Invoke-AzRest -Path "$vmId/deallocate?api-version=$api" -Method POST
if ($resp.StatusCode -lt 200 -or $resp.StatusCode -ge 300) {
  throw "Falha ao desligar (deallocate) VM. HTTP $($resp.StatusCode) $($resp.Content)"
}
Write-Output "Deallocate acionado (202/200)."
```

**Observações úteis:**  
- **Deallocate** libera a computação (e o IP dinâmico), mas mantém o disco.  

---

### 6. Criar Schedules

Crie dois agendamentos para ligar e desligar a VM:

**A) Start-Weekdays-0730**  
- Hora: **07:30**  
- Time zone: **(UTC-03:00) São Paulo** (ou *E. South America Standard Time*)  
- Recurrence: **Daily** (todos os dias) ou **Weekly** (Seg–Sex) → escolha conforme necessidade  
- Expiração: **Sem expiração** (ou defina data limite se for POC)  

**B) Stop-Weekdays-1830**  
- Hora: **18:30**  
- Time zone: **(UTC-03:00) São Paulo** (ou *E. South America Standard Time*)  
- Recurrence: **Daily** (todos os dias) ou **Weekly** (Seg–Sex) → escolha conforme necessidade  
- Expiração: **Sem expiração** (ou defina data limite se for POC)  

---

### 7. Vincular Schedules aos Runbooks

No **Azure Portal**:  
- Vá em **Runbooks → Start-VM → Link to schedule → Selecionar Start-Weekdays-0730**.  
- Vá em **Runbooks → Stop-VM → Link to schedule → Selecionar Stop-Weekdays-1830**.  

---

### 8. Testes e validação

- **A)** Com a VM desligada, abra o runbook **Start-VM → Start**. A VM deverá ligar após alguns segundos.  
- **B)** Com a VM ligada, abra o runbook **Stop-VM → Start**. A VM deverá ser desligada após alguns segundos.  

---

### 9. Limpeza do ambiente

- Excluir o **Resource Group** e todos os recursos criados neste lab (mesmo que em outros RGs).  
- Isso mantém o ambiente limpo e sob controle do ponto de vista **financeiro**.  

---


