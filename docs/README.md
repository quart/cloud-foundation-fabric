# Dokumentacja workflow GitHub Actions w FAST

Ta dokumentacja wyjaśnia jak działają workflow GitHub Actions dla każdego stage'u w architekturze FAST (Foundation and Advanced Setup Templates).

## Dokumenty w tej kolekcji

### 1. [Główna dokumentacja - WORKFLOWS_GITHUB_PL.md](./WORKFLOWS_GITHUB_PL.md)
Kompleksowy przewodnik opisujący:
- Architekturę FAST i jej etapy (stages)
- System generowania workflow
- Strukturę szablonów i konfiguracji CI/CD
- Szczegółowy opis workflow GitHub Actions
- Konfigurację Service Accounts i Workload Identity Federation
- Wykorzystanie w praktyce i bezpieczeństwo

### 2. [Architektura workflow - WORKFLOW_ARCHITECTURE.md](./WORKFLOW_ARCHITECTURE.md)
Diagramy i wizualizacje pokazujące:
- Architekturę systemu CI/CD w FAST
- Przepływ danych między etapami
- Strukturę uprawnień i zabezpieczeń
- Sekwencje działań w workflow

### 3. [Praktyczny przykład - WORKFLOW_EXAMPLE_PL.md](./WORKFLOW_EXAMPLE_PL.md)
Kompletny przykład implementacji:
- Dodanie nowego stage'u "3-app-dev"
- Konfiguracja CI/CD krok po kroku
- Tworzenie i konfiguracja repository GitHub
- Testowanie i troubleshooting workflow

## Szybki start

Jeśli chcesz szybko zrozumieć jak działają workflow w FAST:

1. **Zacznij od [głównej dokumentacji](./WORKFLOWS_GITHUB_PL.md)** - przeczytaj sekcje "Przegląd architektury" i "System generowania workflow"

2. **Sprawdź [diagramy architektury](./WORKFLOW_ARCHITECTURE.md)** - wizualizacje pomogą zrozumieć przepływ danych

3. **Przeanalizuj [praktyczny przykład](./WORKFLOW_EXAMPLE_PL.md)** - zobacz jak w praktyce skonfigurować nowy stage

## Kluczowe koncepcje

### Etapy FAST (Stages)
- **Stage 0 (Bootstrap)** - Podstawowa konfiguracja organizacji
- **Stage 1 (Resource Management)** - Hierarchia zasobów i automatyzacja
- **Stage 2** - Zasoby współdzielone (networking, security)
- **Stage 3** - Zasoby środowiska (aplikacje, workloady)

### Komponenty CI/CD
- **Szablony workflow** - `templates/workflow-github.yaml`
- **Konfiguracja CI/CD** - `cicd.tf`, `stage-cicd.tf`
- **Generowanie** - `outputs-cicd.tf`
- **Deployment** - `fast/extras/0-cicd-github`

### Bezpieczeństwo
- **Workload Identity Federation** - Bezpieczne uwierzytelnianie bez kluczy
- **Service Accounts RO/RW** - Oddzielne uprawnienia dla plan/apply
- **Minimalne uprawnienia** - Każdy stage ma tylko potrzebne uprawnienia

## Często zadawane pytania

### Jak dodać nowy stage?
Zobacz [praktyczny przykład](./WORKFLOW_EXAMPLE_PL.md) - sekcja "Scenariusz: Dodanie nowego stage".

### Jak skonfigurować różne środowiska?
W konfiguracji stage'u ustaw odpowiedni `environment` (dev/prod) i `env` - workflow automatycznie użyje odpowiednich Service Accounts.

### Jak działa uwierzytelnianie?
System używa Workload Identity Federation - GitHub Actions token jest wymieniany na Google Cloud access token bez potrzeby przechowywania kluczy.

### Jak troubleshootować problemy z workflow?
1. Sprawdź logi w GitHub Actions
2. Zweryfikuj konfigurację Service Accounts
3. Sprawdź uprawnienia Workload Identity Federation
4. Zweryfikuj dostępność Cloud Storage buckets

## Wsparcie i rozwój

Ten system jest częścią projektu Google Cloud Foundation Fabric. Aby uzyskać więcej informacji:

- Sprawdź [główne README](../README.md) projektu
- Zobacz [dokumentację FAST stages](../fast/stages/README.md)
- Przeczytaj [contributing guidelines](../CONTRIBUTING.md)

## Struktura plików

```
docs/
├── README.md                    # Ten plik
├── WORKFLOWS_GITHUB_PL.md      # Główna dokumentacja
├── WORKFLOW_ARCHITECTURE.md    # Diagramy architektury
└── WORKFLOW_EXAMPLE_PL.md      # Praktyczny przykład

fast/stages/
├── 0-bootstrap/
│   ├── cicd.tf                 # Konfiguracja CI/CD
│   └── templates/
│       └── workflow-github.yaml # Szablon workflow
├── 1-resman/
│   ├── stage-cicd.tf           # Konfiguracja CI/CD
│   ├── outputs-cicd.tf         # Generowane workflow
│   └── templates/
│       └── workflow-github.yaml # Szablon workflow
└── ...

fast/extras/0-cicd-github/       # Automatyzacja GitHub repositories
```

Ta dokumentacja powinna dostarczyć wszystkich informacji potrzebnych do zrozumienia i pracy z workflow GitHub Actions w architekturze FAST.