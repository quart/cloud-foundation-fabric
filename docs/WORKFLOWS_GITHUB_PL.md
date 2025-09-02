# Jak działają workflow GitHub Actions dla każdego stage w FAST

Ten dokument opisuje jak implementowane są workflow GitHub Actions dla każdego etapu (stage) w architekturze FAST (Foundation and Advanced Setup Templates) w repozytorium cloud-foundation-fabric.

## Przegląd architektury

FAST implementuje wieloetapowe wdrażanie infrastruktury Google Cloud Platform używając Terraform. Każdy etap (stage) może być uruchamiany niezależnie lub jako część większego wdrożenia organizacyjnego.

### Główne etapy:

1. **Stage 0 (Bootstrap)** - Konfiguracja podstawowa organizacji i automatyzacji
2. **Stage 1 (Resource Management)** - Zarządzanie hierarchią zasobów i folderami
3. **Stage 2** - Zasoby współdzielone (networking, security, project factory)
4. **Stage 3** - Zasoby poziomu środowiska (GKE, GCVE, etc.)

## System generowania workflow

### 1. Struktura szablonów

Każdy stage zawiera szablony workflow w katalogu `templates/`:

```
fast/stages/{stage}/templates/
├── workflow-github.yaml     # Szablon dla GitHub Actions
├── workflow-gitlab.yaml     # Szablon dla GitLab CI
└── providers.tf.tpl         # Szablon dostawców Terraform
```

### 2. Konfiguracja CI/CD

#### Stage 0 (Bootstrap) - `cicd.tf`

```hcl
locals {
  _cicd_configs = merge(
    # konfiguracja dla stage'ów
    {
      for k, v in var.cicd_config : k => merge(v, {
        level = k == "bootstrap" ? 0 : 1
        stage = k
      }) if v != null
    },
    # konfiguracja dla dodatków (addons)
    {
      for k, v in var.fast_addon : k => merge(v.cicd_config, {
        level = 1
        stage = substr(v.parent_stage, 2, -1)
      }) if v.cicd_config != null
    }
  )
}
```

#### Stage 1 (Resource Management) - `stage-cicd.tf`

```hcl
locals {
  _cicd_configs = merge(
    # stage 2
    {
      for k, v in local.stage2 : k => merge(v.cicd_config, {
        env        = "prod"
        level      = 2
        stage      = replace(k, "_", "-")
        short_name = v.short_name
      }) if v.cicd_config != null
    },
    # stage 3
    {
      for k, v in local.stage3 : k => merge(v.cicd_config, {
        env        = v.environment
        level      = 3
        short_name = coalesce(v.short_name, k)
        stage      = replace(k, "_", "-")
      }) if v.cicd_config != null
    }
  )
}
```

### 3. Generowanie workflow

Workflow są generowane jako outputs w plikach `outputs-cicd.tf`:

```hcl
locals {
  cicd_workflows = {
    for k, v in local.cicd_repositories : "${v.level}-${replace(k, "_", "-")}" => templatefile(
      "${path.module}/templates/workflow-${v.repository.type}.yaml", {
        audiences = try(
          local.identity_providers[v.identity_provider].audiences, []
        )
        identity_provider = try(
          local.identity_providers[v.identity_provider].name, ""
        )
        outputs_bucket = var.automation.outputs_bucket
        service_accounts = {
          apply = try(module.cicd-sa-rw[k].email, "")
          plan  = try(module.cicd-sa-ro[k].email, "")
        }
        stage_name = k
        tf_providers_files = {
          apply = replace(local.cicd_workflow_providers[k], "_", "-")
          plan  = replace(local.cicd_workflow_providers["${k}-r"], "_", "-")
        }
        tf_var_files = [ /* lista plików tfvars */ ]
      }
    )
  }
}
```

## Struktura workflow GitHub Actions

### Wyzwalacze (Triggers)

```yaml
on:
  pull_request:
    branches:
      - main
    types:
      - closed      # PR zamknięte (dla apply)
      - opened      # PR otwarte (dla plan)
      - synchronize # Aktualizacje PR (dla plan)
```

### Zmienne środowiskowe

```yaml
env:
  FAST_SERVICE_ACCOUNT: ${service_accounts.apply}      # SA dla apply
  FAST_SERVICE_ACCOUNT_PLAN: ${service_accounts.plan}  # SA dla plan
  FAST_WIF_PROVIDER: ${identity_provider}              # Workload Identity Federation
  TF_PROVIDERS_FILE: ${tf_providers_files.apply}       # Plik dostawców dla apply
  TF_PROVIDERS_FILE_PLAN: ${tf_providers_files.plan}   # Plik dostawców dla plan
  TF_VERSION: 1.11.4                                   # Wersja Terraform
```

### Główne kroki workflow

#### 1. Checkout i konfiguracja SSH

```yaml
- name: Checkout repository
  uses: actions/checkout@v4

- name: Configure SSH authentication
  run: |
    ssh-agent -a "$SSH_AUTH_SOCK" > /dev/null
    ssh-add - <<< "${{ secrets.CICD_MODULES_KEY }}"
```

#### 2. Konfiguracja zmiennych dla plan/apply

```yaml
# Dla operacji plan (PR otwarte/aktualizowane)
- name: Set up plan variables
  if: github.event.pull_request.merged != true && success()
  run: |
    echo "plan_opts=-lock=false" >> "$GITHUB_ENV"
    echo "provider_file=${{env.TF_PROVIDERS_FILE_PLAN}}" >> "$GITHUB_ENV"
    echo "service_account=${{env.FAST_SERVICE_ACCOUNT_PLAN}}" >> "$GITHUB_ENV"

# Dla operacji apply (PR zamknięte i merged)
- name: Set up apply variables
  if: github.event.pull_request.merged == true && success()
  run: |
    echo "provider_file=${{env.TF_PROVIDERS_FILE}}" >> "$GITHUB_ENV"
    echo "service_account=${{env.FAST_SERVICE_ACCOUNT}}" >> "$GITHUB_ENV"
```

#### 3. Uwierzytelnianie w Google Cloud

```yaml
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{env.FAST_WIF_PROVIDER}}
    service_account: ${{env.service_account}}
    access_token_lifetime: 900s

- name: Set up Cloud SDK
  uses: google-github-actions/setup-gcloud@v2
  with:
    install_components: alpha
```

#### 4. Pobieranie konfiguracji Terraform

```yaml
- name: Copy Terraform provider file
  run: |
    gcloud storage cp -r \
      "gs://${outputs_bucket}/providers/${{env.provider_file}}" ./
    # Pobieranie plików tfvars dla każdego stage
    gcloud storage cp -r \
      "gs://${outputs_bucket}/tfvars/0-bootstrap.auto.tfvars.json" ./
```

#### 5. Wykonanie Terraform

```yaml
- name: Terraform init
  run: terraform init -no-color

- name: Terraform validate
  run: terraform validate -no-color

- name: Terraform plan
  run: terraform plan -input=false -out ../plan.out -no-color ${{env.plan_opts}}

- name: Terraform apply
  if: github.event.pull_request.merged == true && success()
  run: terraform apply -input=false -auto-approve -no-color ../plan.out
```

#### 6. Komentarze w Pull Request

Workflow automatycznie dodaje komentarze do PR z wynikami operacji Terraform:

```yaml
- name: Post comment to Pull Request
  uses: actions/github-script@v7
  if: github.event_name == 'pull_request'
  with:
    script: |
      const output = `### Terraform Initialization \`${{steps.tf-init.outcome}}\`
      ### Terraform Validation \`${{steps.tf-validate.outcome}}\`
      ### Terraform Plan \`${{steps.tf-plan.outcome}}\`
      ### Terraform Apply \`${{steps.tf-apply.outcome}}\``;
      
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: output
      })
```

## Konta usług (Service Accounts)

### Dwa typy kont usług

1. **Read-Write (RW)** - do operacji apply
2. **Read-Only (RO)** - do operacji plan

### Konfiguracja w Stage 1

```hcl
module "cicd-sa-rw" {
  source     = "../../../modules/iam-service-account"
  for_each   = local.cicd_repositories
  project_id = var.automation.project_id
  name = templatestring(var.resource_names["sa-cicd_rw"], {
    name = each.value.short_name
  })
  display_name = "CI/CD ${each.value.level}-${each.value.short_name} ${each.value.env} service account."
  prefix = "${var.prefix}-${var.environments[each.value.env].short_name}"
  iam = {
    "roles/iam.workloadIdentityUser" = [
      # Konfiguracja Workload Identity Federation
    ]
  }
}
```

## Workload Identity Federation

### Konfiguracja dostawców tożsamości

```hcl
cicd_providers = {
  for k, v in google_iam_workload_identity_pool_provider.default :
  k => {
    audiences = concat(
      v.oidc[0].allowed_audiences,
      ["https://iam.googleapis.com/${v.name}"]
    )
    issuer           = local.workload_identity_providers[k].issuer
    issuer_uri       = try(v.oidc[0].issuer_uri, null)
    name             = v.name
    principal_branch = local.workload_identity_providers[k].principal_branch
    principal_repo   = local.workload_identity_providers[k].principal_repo
  }
}
```

## Wykorzystanie workflow w praktyce

### 1. Konfiguracja repozytoriów GitHub

Moduł `fast/extras/0-cicd-github` automatyzuje:
- Tworzenie repozytoriów GitHub
- Wypełnianie początkowych plików
- Konfiguracja kluczy SSH dla dostępu do modułów
- Tworzenie Pull Requestów z konfiguracją

### 2. Przykład konfiguracji

```hcl
repositories = {
  fast_00_bootstrap = {
    create_options = {
      description = "FAST bootstrap."
      features = {
        issues = true
      }
    }
    populate_from = "../../stages/0-bootstrap"
  }
  fast_01_resman = {
    create_options = {
      description = "FAST resource management."
    }
    populate_from = "../../stages/1-resman"
  }
}
```

### 3. Automatyczne wdrażanie

Po skonfigurowaniu:
1. Otwarcie PR → uruchamia `terraform plan`
2. Aktualizacja PR → ponownie `terraform plan`
3. Merge PR → uruchamia `terraform apply`
4. Komentarze w PR zawierają wyniki operacji

## Bezpieczeństwo

### Uprawnienia minimalne

- Konta RO mają tylko uprawnienia do odczytu
- Konta RW mają uprawnienia tylko do zarządzania zasobami swojego stage'u
- Workload Identity Federation zapewnia bezpieczne uwierzytelnianie bez kluczy

### Separacja środowisk

- Różne konta usług dla różnych środowisk
- Osobne buckets dla output'ów każdego stage'u
- Kontrola dostępu na poziomie IAM

## Rozszerzenia

System można rozszerzyć o:
- Dodatkowe stage'y poprzez konfigurację w `local.stage3`
- Vlastne addony w `var.fast_addon`
- Dodatkowe dostawców CI/CD (np. GitLab)
- Dodatkowe kroki walidacji w workflow

Ten system zapewnia spójne, bezpieczne i skalowalne podejście do CI/CD dla całej infrastruktury Google Cloud Platform.