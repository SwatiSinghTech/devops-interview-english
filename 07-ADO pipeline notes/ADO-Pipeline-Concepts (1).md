# Azure DevOps Pipeline — Concepts (Basic to Advanced)

Ye notes Azure DevOps pipeline ke saare concepts cover karte hain — **basic se advance** tak.
Pehle samjho pipeline ke andar kya hota hai, phir wo kahaan chalti hai, aur phir extra controls.

---

## 1. Pipeline ka dhaancha (Basic)

Pipeline ek **automation workflow** hai. Andar ka structure aisa hai — bade se chhota:

- **Pipeline** — pura automation workflow.
- **Stage** — bada phase, jaise Build, Scanning, Deploy.
- **Job** — steps ka group; ek agent par chalta hai. Ek stage mein kai jobs ho sakte hain.
- **Step / Task** — sabse chhoti cheez; ek single command (jaise `terraform init`).

> **Note:** Agar YAML mein sirf steps likho (job na likho), tab bhi wo ek default job ke andar, ek agent par hi chalte hain. Matlab job hamesha hota hai.

```
Pipeline
└── Stage (Build / Scanning / Deploy)
    └── Job (ek agent par chalta hai)
        └── Step / Task (ek command)
```

---

## 2. Agent aur Pool (Execution)

Yahaan samjho job **kahaan** chalti hai.

- **Agent** — wo machine (VM / server / container) jahaan job actually chalti hai.
- **Pool** — agents ka group. YAML mein `pool: self-hosted-pool` likhte hain.

**Pool override rule:** job-level pool pipeline-level pool ko **override** kar deta hai.
Agar job par koi pool na ho, to wo job pipeline-level wale pool par chalta hai.

**`server` pool:** Manual Validation jaisa **agentless** job kisi machine par nahi chalta,
isliye uska pool `server` rakhte hain.

### Agent ki do type

- **Microsoft-hosted** — Microsoft ki ready machine; aasan, maintenance-free.
- **Self-hosted** — apni machine; zyada **control** aur **security** (custom tools install kar sakte ho).

---

## 3. Advanced Concepts

### Trigger
Pipeline **kab** chalu ho. Jis branch par code aane se chalani hai, use `include` mein dete hain.

```yaml
trigger:
  branches:
    include:
      - main
      - feature/*
```

### Parallelism
Bina dependency wale jobs **ek saath (parallel)** chalte hain — time bachta hai.
Parallel chalane ke liye jobs ke beech `dependsOn` mat lagao.

### dependsOn
Jobs ya stages ka **order** set karne ke liye. Ek tabhi chale jab doosra khatam ho.

```yaml
- job: TerraformApply
  dependsOn: ManualValidation
```

### Parameters
Pipeline **chalate waqt** diya jaane wala input. Jaise environment `dev` ya `prod`.

```yaml
parameters:
  - name: environment
    type: string
    values:
      - dev
      - prod
```

### Variables
Pipeline ke **andar set** ki gayi values. Jaise working directory parameter ke hisaab se:

```yaml
variables:
  workDir: '$(System.DefaultWorkingDirectory)/environments/${{ parameters.environment }}'
```

### Conditions
Kaam **tabhi** chale jab koi shart sach ho. Jaise Deploy sirf `main` branch par:

```yaml
condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
```

### Manual Approval
Production se pehle **insaan ka final check**. Manual Validation task ya Environment approval se,
taaki galat cheez live na ho. Iska job `pool: server` par chalta hai (agentless).

---

## 4. Ek nazar mein (summary)

| Concept | Ek line mein |
|---------|--------------|
| **Pipeline** | Pura automation workflow |
| **Stage** | Bada phase (Build / Scanning / Deploy) |
| **Job** | Steps ka group; agent par chalta hai |
| **Step / Task** | Ek single command |
| **Agent** | Wo machine jahaan job chalti hai |
| **Pool** | Agents ka group |
| **Trigger** | Pipeline kab chale (kis branch par) |
| **Parallelism** | Bina dependency wale jobs ek saath |
| **dependsOn** | Jobs / stages ka order |
| **Parameters** | Chalate waqt input (dev / prod) |
| **Variables** | Andar set ki values |
| **Conditions** | Shart sach ho tabhi chale |
| **Manual Approval** | Production se pehle insaan ka check |

---

## 5. Chhota sample (sab ek saath)

```yaml
trigger:
  branches:
    include:
      - main
      - feature/*

pool: self-hosted-pool

parameters:
  - name: environment
    type: string
    default: dev
    values:
      - dev
      - prod

variables:
  workDir: '$(System.DefaultWorkingDirectory)/environments/${{ parameters.environment }}'

stages:
  - stage: Build
    jobs:
      - job: FmtValidate
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: 'latest'
          - task: TerraformTask@5
            inputs:
              provider: 'azurerm'
              command: 'validate'
              workingDirectory: $(workDir)

  - stage: Deploy
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
    jobs:
      - job: ManualValidation
        pool: server
        steps:
          - task: ManualValidation@1
            inputs:
              notifyUsers: 'team-lead@company.com'
      - job: TerraformApply
        dependsOn: ManualValidation
        steps:
          - task: TerraformTask@5
            inputs:
              provider: 'azurerm'
              command: 'apply'
              workingDirectory: $(workDir)
```
