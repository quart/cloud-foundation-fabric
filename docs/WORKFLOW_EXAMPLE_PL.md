# Przykład konfiguracji workflow dla nowego stage

Ten dokument przedstawia praktyczny przykład jak skonfigurować workflow GitHub Actions dla nowego stage w architekturze FAST.

## Scenariusz: Dodanie nowego stage "3-app-dev"

Załóżmy, że chcemy dodać nowy stage do wdrażania aplikacji deweloperskich.

### 1. Konfiguracja w Stage 1 (Resource Management)

W pliku `fast/stages/1-resman/variables.tf` dodajemy konfigurację dla nowego stage:

```hcl
# W sekcji local.stage3
variable "stage3" {
  description = "Stage 3 configurations."
  type = map(object({
    # ... existing configuration ...
    
    # Nowa konfiguracja dla app-dev
    app_dev = optional(object({
      cicd_config = optional(object({
        identity_provider = string
        repository = object({
          name   = string
          type   = string
          branch = optional(string)
        })
      }))
      environment = optional(string, "dev")
      short_name  = optional(string, "app-dev")
    }))
  }))
}
```

### 2. Konfiguracja CI/CD w terraform.tfvars

```hcl
stage3 = {
  app_dev = {
    cicd_config = {
      identity_provider = "github"
      repository = {
        name   = "fast-app-dev"
        type   = "github"
        branch = "main"
      }
    }
    environment = "dev"
    short_name  = "app-dev"
  }
}

# Konfiguracja Workload Identity Provider
workload_identity_providers = {
  github = {
    issuer = "https://token.actions.githubusercontent.com"
    custom_settings = {
      audiences = ["https://github.com/my-org"]
    }
  }
}
```

### 3. Utworzenie struktury stage'u

```bash
# Tworzymy nowy katalog stage'u
mkdir -p fast/stages/3-app-dev/templates

# Kopiujemy podstawowe pliki z innego stage'u
cp fast/stages/3-gke-dev/templates/workflow-github.yaml fast/stages/3-app-dev/templates/
cp fast/stages/3-gke-dev/templates/providers.tf.tpl fast/stages/3-app-dev/templates/
```

### 4. Dostosowanie szablonu workflow

W pliku `fast/stages/3-app-dev/templates/workflow-github.yaml`:

```yaml
name: "FAST ${stage_name} application development stage"

on:
  pull_request:
    branches:
      - main
    types:
      - closed
      - opened
      - synchronize
  # Dodatkowy trigger dla deployment tagów
  push:
    tags:
      - 'deploy-*'

env:
  FAST_SERVICE_ACCOUNT: ${service_accounts.apply}
  FAST_SERVICE_ACCOUNT_PLAN: ${service_accounts.plan}
  FAST_WIF_PROVIDER: ${identity_provider}
  SSH_AUTH_SOCK: /tmp/ssh_agent.sock
  TF_PROVIDERS_FILE: ${tf_providers_files.apply}
  TF_PROVIDERS_FILE_PLAN: ${tf_providers_files.plan}
  TF_VERSION: 1.11.4
  # Dodatkowe zmienne dla aplikacji
  APP_ENVIRONMENT: development
  DEPLOY_REGION: europe-west1

jobs:
  fast-pr:
    if: >-
      github.event.action == 'closed' && 
      github.event.pull_request.merged == true ||
      github.event.action == 'opened' ||
      github.event.action == 'synchronize'
    permissions:
      contents: read
      id-token: write
      issues: write
      pull-requests: write
    runs-on: ubuntu-latest
    steps:
      # ... standardowe kroki ...
      
      # Dodatkowy krok: walidacja aplikacji
      - id: app-validate
        name: Validate application configuration
        run: |
          # Sprawdzenie czy wymagane pliki aplikacji istnieją
          if [[ ! -f "app.yaml" ]]; then
            echo "Error: app.yaml not found"
            exit 1
          fi
          
          # Walidacja YAML
          python -c "import yaml; yaml.safe_load(open('app.yaml'))"
          
      # Dodatkowy krok: security scanning
      - id: security-scan
        name: Security scan
        if: github.event.pull_request.merged != true
        run: |
          # Skanowanie bezpieczeństwa przed aplikacją
          terraform plan -out=plan.tfplan
          # tfsec . || true  # Opcjonalne: nie przerywaj na błędach
```

### 5. Konfiguracja outputs dla nowego stage

W `fast/stages/1-resman/outputs-cicd.tf` zostanie automatycznie dodany output dla nowego stage dzięki pętli:

```hcl
# To zostanie automatycznie wygenerowane dla app_dev
output "cicd_workflows" {
  description = "CI/CD workflow definitions."
  value = {
    for k, v in local.cicd_workflows : k => {
      name     = "${v.level}-${replace(k, "_", "-")}"
      content  = v.content
      filename = "${v.level}-${replace(k, "_", "-")}.yaml"
    }
  }
}
```

### 6. Utworzenie repository za pomocą extras/0-cicd-github

W `fast/extras/0-cicd-github/terraform.tfvars`:

```hcl
organization = "my-github-org"

# Konfiguracja modułów (jeśli używasz prywatnego repo)
modules_config = {
  repository_name = "my-org/cloud-foundation-fabric"
  key_config = {
    create_key     = true
    create_secrets = true
  }
}

# Konfiguracja repozytoriów
repositories = {
  fast-app-dev = {
    create_options = {
      description = "FAST application development stage"
      visibility  = "private"
      features = {
        issues   = true
        projects = false
        wiki     = false
      }
      allow = {
        auto_merge     = true
        merge_commit   = true
        rebase_merge   = true
        squash_merge   = true
      }
    }
    populate_from = "../../stages/3-app-dev"
  }
}

# Konfiguracja commit'ów
commit_config = {
  author  = "FAST Automation"
  email   = "fast-automation@my-org.com"
  message = "Initial FAST stage configuration"
}

# Konfiguracja Pull Request'u
pull_request_config = {
  create   = true
  title    = "Initial FAST app-dev stage setup"
  body     = "Automated setup of FAST application development stage"
  base_ref = "main"
}
```

### 7. Uruchomienie konfiguracji

```bash
# 1. Aktualizuj stage 1 (Resource Management)
cd fast/stages/1-resman
terraform plan
terraform apply

# 2. Pobierz wygenerowane workflow
gcloud storage cp gs://BUCKET/cicd-workflows/3-app-dev.yaml ./

# 3. Skonfiguruj repository GitHub
cd ../../extras/0-cicd-github
terraform plan
terraform apply

# 4. Wypchaj konfigurację do nowego repo
git clone https://github.com/my-org/fast-app-dev.git
cd fast-app-dev
cp ../path/to/3-app-dev.yaml .github/workflows/
git add .
git commit -m "Add CI/CD workflow"
git push
```

### 8. Testowanie workflow

1. **Utwórz branch feature**:
```bash
git checkout -b feature/add-new-app
# Dodaj pliki aplikacji
echo "apiVersion: v1" > app.yaml
git add app.yaml
git commit -m "Add application configuration"
git push -u origin feature/add-new-app
```

2. **Otwórz Pull Request**:
   - Workflow automatycznie uruchomi `terraform plan`
   - Dodane zostaną komentarze z wynikami planowania

3. **Sprawdź wyniki**:
   - Sprawdź czy plan Terraform jest poprawny
   - Sprawdź czy walidacja aplikacji przeszła
   - Sprawdź wyniki skanowania bezpieczeństwa

4. **Merguj PR**:
   - Po mergeu workflow uruchomi `terraform apply`
   - Infrastruktura zostanie wdrożona

### 9. Monitoring i troubleshooting

```bash
# Sprawdzenie statusu workflow
gh run list --repo my-org/fast-app-dev

# Szczegóły konkretnego run'u
gh run view <run-id> --repo my-org/fast-app-dev

# Logi z konkretnego job'a
gh run view <run-id> --log --repo my-org/fast-app-dev
```

## Najlepsze praktyki

1. **Testowanie lokalne**:
```bash
# Test szablonu workflow przed wdrożeniem
terraform plan -var-file=test.tfvars -out=test.plan
terraform show -json test.plan | jq '.resource_changes'
```

2. **Bezpieczeństwo**:
   - Używaj różnych Service Account dla plan/apply
   - Ogranicz uprawnienia do minimum
   - Regularnie przeglądaj access logi

3. **Wersjonowanie**:
   - Taguj wersje infrastructure
   - Używaj semantic versioning
   - Dokumentuj breaking changes

4. **Backup i disaster recovery**:
   - Backup Terraform state regularnie
   - Testuj procedury odzyskiwania
   - Dokumentuj procedury rollback

Ten przykład pokazuje kompletny proces dodania nowego stage'u z automatycznym CI/CD workflow w architekturze FAST.