# Laboratorio 3. Entrega continua y despliegue con servicios en la nube

**Modalidad:** parejas  
**Valor:** 5 % de la calificación final  
**Repositorio:** se continúa trabajando sobre el mismo repositorio utilizado en el Laboratorio 2  
**Plataformas:** GitHub Actions, Google Cloud Artifact Registry, Cloud Run y Firestore  
**Entrega:** repositorio privado en GitHub con acceso para ambos integrantes de la pareja y para `guillermocalderon2021`

## 1. Propósito

En el Laboratorio 2 se construyó un pipeline de integración continua capaz de validar cambios mediante lint, pruebas y construcción de la imagen de contenedor. En este laboratorio el repositorio se extiende hasta completar un proceso de entrega continua.

El objetivo es que un cambio que ya atravesó los controles de integración continua pueda:

1. producir una imagen identificable;
2. publicar esa imagen en un registry;
3. desplegar exactamente ese mismo artefacto como una revisión candidata;
4. verificar la revisión antes de exponerla al tráfico principal;
5. promoverla de forma explícita;
6. recuperar una revisión anterior sin reconstruir la aplicación.

La solución debe conservar trazabilidad entre el commit de Git, la imagen almacenada en Artifact Registry, su digest y la revisión creada en Cloud Run.

## 2. Resultado esperado

Al terminar el laboratorio, el flujo deberá corresponder al siguiente modelo:

```text
Pull request hacia main
        │
        ├── Lint
        ├── Test
        └── Container
                │
                └── no publica ni despliega
                         │
                         ▼
                 merge / push a main
                         │
                         ▼
              imagen del commit
                         │
                         ▼
                Artifact Registry
                         │
                  tag = commit SHA
                  digest = sha256:...
                         │
                         ▼
          revisión candidata en Cloud Run
                0 % del tráfico
                tag de tráfico: candidate
                         │
                  /health + /incidents
                         │
                         ▼
               promoción manual
                         │
                         ▼
                 100 % del tráfico
                         │
                         ▼
                 rollback manual
                         │
                         ▼
              revisión anterior, 100 %
```

Cloud Run debe utilizar **un solo servicio**, denominado `incident-api`. Las versiones desplegadas de la aplicación se representarán mediante revisiones del mismo servicio.

## 3. Reglas técnicas

La solución deberá cumplir las siguientes reglas:

- `main` continúa protegida mediante el ruleset configurado en el Laboratorio 2.
- Los checks `Lint`, `Test` y `Container` continúan siendo obligatorios antes de integrar un pull request.
- Un pull request no debe publicar imágenes ni desplegar recursos.
- La publicación y el despliegue automático solamente deben ocurrir después de un `push` a `main`.
- La imagen publicada debe utilizar el **SHA completo del commit** como tag.
- La imagen que se despliega debe identificarse mediante su **digest**, no reconstruirse.
- La revisión candidata debe crearse con **0 % del tráfico principal**.
- La revisión candidata debe ser accesible mediante un **traffic tag** llamado `candidate`.
- La revisión candidata debe verificarse con `GET /health` y `GET /incidents`.
- La promoción a 100 % debe requerir una acción manual.
- El rollback debe requerir una acción manual y debe mover el tráfico hacia una revisión anterior existente.
- El rollback no puede ejecutar `docker build`, crear otra imagen ni volver a publicar una versión anterior.
- GitHub Actions debe autenticarse en Google Cloud mediante OIDC y Workload Identity Federation.
- No se permite crear ni almacenar una llave JSON de una service account.
- Deben utilizarse identidades separadas para publicación, despliegue y ejecución de la aplicación.
- La aplicación desplegada debe utilizar Firestore mediante `STORAGE_BACKEND=firestore`.
- No deben crearse servicios adicionales de Cloud Run para representar staging, candidate o producción.

## 4. Recursos que se utilizarán

Cada pareja utilizará un proyecto de Google Cloud dedicado al laboratorio.

| Recurso | Nombre |
| --- | --- |
| Región | `us-central1` |
| Artifact Registry | `devops-images` |
| Imagen | `incident-api` |
| Firestore | `(default)` |
| Servicio Cloud Run | `incident-api` |
| Revisión inicial | `incident-api-baseline` |
| Traffic tag de candidato | `candidate` |
| Workload Identity Pool | `github-lab3` |
| Provider | `github` |
| Service account de publicación | `gha-publisher` |
| Service account de despliegue | `gha-deployer` |
| Service account de ejecución | `incident-api-runtime` |

La separación de responsabilidades es:

```text
gha-publisher
└── publicar imágenes en devops-images

gha-deployer
├── leer imágenes de devops-images
├── crear nuevas revisiones de incident-api
├── modificar el tráfico de incident-api
└── ejecutar revisiones con incident-api-runtime

incident-api-runtime
└── acceder a Firestore
```

## 5. Antes de comenzar

### 5.1 Estado del Laboratorio 2

Antes de modificar el pipeline se debe comprobar que:

1. el repositorio utilizado en el Laboratorio 2 se encuentra actualizado;
2. `main` no contiene cambios locales pendientes;
3. el último workflow de integración continua finaliza correctamente;
4. el ruleset de `main` continúa activo;
5. ambos integrantes y el docente tienen acceso al repositorio privado.

Comprobación local:

```bash
git status
git branch --show-current
git pull
```

El bootstrap debe ejecutarse desde la raíz del repositorio, donde deben existir al menos:

```text
Dockerfile
app/
requirements.txt
.github/
```

### 5.2 Entorno local

En Linux se utilizará una terminal Bash.

En Windows se recomienda utilizar **WSL2 con Ubuntu**. El script de preparación no está diseñado para ejecutarse directamente en PowerShell ni en `cmd.exe`.

Antes de continuar:

```bash
gcloud --version
docker --version
git --version
curl --version
```

También debe comprobarse que Docker se encuentra en ejecución:

```bash
docker info
```

### 5.3 Proyecto de Google Cloud

Cada pareja debe crear un proyecto de Google Cloud destinado al laboratorio y habilitar una cuenta de facturación.

No se debe reutilizar un proyecto que ya contenga una base Firestore `(default)` en otra región o un servicio Cloud Run llamado `incident-api`.

Autenticación local:

```bash
gcloud auth login
```

Comprobación:

```bash
gcloud auth list
```

La cuenta utilizada para ejecutar el bootstrap debe tener permisos suficientes para habilitar APIs, crear recursos, administrar IAM y crear la base Firestore.

## 6. Control de costos

### 6.1 Presupuesto

Antes de ejecutar el bootstrap se debe crear manualmente un presupuesto para el proyecto:

1. abrir **Google Cloud Console**;
2. ingresar a **Billing**;
3. abrir **Budgets & alerts**;
4. seleccionar **Create budget**;
5. limitar el alcance al proyecto utilizado en el laboratorio;
6. establecer un presupuesto mensual de **USD 1.00**;
7. configurar alertas de gasto real al **50 %**, **90 %** y **100 %**.

Un presupuesto y sus alertas **no detienen automáticamente el gasto**. Su función es advertir que el consumo está alcanzando los límites configurados.

Conservar una captura donde se observe el presupuesto y el proyecto al que está asociado. Esta evidencia se incorporará a `EVIDENCIAS.md`.

### 6.2 Controles incluidos en el laboratorio

El bootstrap configura:

- Artifact Registry y Cloud Run en `us-central1`;
- una cleanup policy en Artifact Registry;
- protección de la versión con tag `baseline`;
- conservación de las tres versiones más recientes;
- eliminación periódica de versiones antiguas no protegidas;
- vulnerability scanning deshabilitado para este laboratorio;
- Firestore `(default)`;
- Cloud Run con mínimo de instancias igual a `0` y máximo igual a `2`.

No se debe habilitar Cloud SQL, VPC Connector, Load Balancer, GKE ni servicios adicionales para completar este laboratorio.

## 7. Preparación automatizada del entorno

La siguiente preparación forma parte del laboratorio, pero el script es suministrado. La responsabilidad de la pareja consiste en:

- ejecutarlo con los valores correctos;
- verificar el estado resultante;
- conservar evidencia;
- comprender qué recursos e identidades fueron creados.

### 7.1 Variables

Desde la raíz del repositorio, definir:

```bash
export PROJECT_ID="ID_DEL_PROYECTO_GCP"
export REGION="us-central1"
export GITHUB_OWNER="PROPIETARIO_DEL_REPOSITORIO"
export GITHUB_REPOSITORY="NOMBRE_DEL_REPOSITORIO"
```

Ejemplo:

```bash
export PROJECT_ID="devops-pareja-07"
export REGION="us-central1"
export GITHUB_OWNER="estudiante01"
export GITHUB_REPOSITORY="devops-cicd"
```

Comprobar antes de continuar:

```bash
printf '%s\n' "$PROJECT_ID"
printf '%s\n' "$REGION"
printf '%s\n' "$GITHUB_OWNER"
printf '%s\n' "$GITHUB_REPOSITORY"
```

### 7.2 Crear el archivo

En la raíz del repositorio crear:

```text
bootstrap_lab3.sh
```

con el siguiente contenido.

> El script puede volver a ejecutarse cuando una operación haya fallado parcialmente. Reutiliza los recursos que ya existen. Además, incorpora reintentos para las operaciones IAM que pueden fallar temporalmente mientras Google Cloud propaga una identidad recién creada.

```bash
#!/usr/bin/env bash
set -euo pipefail

# ============================================================================
# Laboratorio 3 - DevOps ESEN
# Bootstrap de Google Cloud para Incident API
#
# Preparación del entorno:
# - Artifact Registry y cleanup policy
# - Firestore (default)
# - identidades separadas para publicación, despliegue y runtime
# - Workload Identity Federation para GitHub Actions
# - revisión baseline de Cloud Run
# - validación de /health y /incidents
#
# Debe ejecutarse desde la raíz del repositorio utilizado en el Laboratorio 2.
# ============================================================================

: "${PROJECT_ID:?Debe definir PROJECT_ID}"
: "${GITHUB_OWNER:?Debe definir GITHUB_OWNER}"
: "${GITHUB_REPOSITORY:?Debe definir GITHUB_REPOSITORY}"

REGION="${REGION:-us-central1}"

AR_REPOSITORY="devops-images"
IMAGE_NAME="incident-api"
RUN_SERVICE="incident-api"

POOL_ID="github-lab3"
PROVIDER_ID="github"

PUBLISHER_SA_NAME="gha-publisher"
DEPLOYER_SA_NAME="gha-deployer"
RUNTIME_SA_NAME="incident-api-runtime"

PUBLISHER_SA="${PUBLISHER_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
DEPLOYER_SA="${DEPLOYER_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
RUNTIME_SA="${RUNTIME_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

GITHUB_REPO="${GITHUB_OWNER}/${GITHUB_REPOSITORY}"
BASELINE_REVISION="${RUN_SERVICE}-baseline"
BASELINE_IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPOSITORY}/${IMAGE_NAME}:baseline"

section() {
  printf '\n============================================================\n'
  printf '%s\n' "$1"
  printf '============================================================\n'
}

fail() {
  printf '\nERROR: %s\n' "$1" >&2
  exit 1
}

require_command() {
  command -v "$1" >/dev/null 2>&1 || fail "No se encontró el comando requerido: $1"
}

# Ejecuta nuevamente un comando que puede fallar temporalmente por propagación
# eventual de IAM. Se utiliza únicamente en operaciones IAM concretas.
retry_iam() {
  local description="$1"
  shift

  local max_attempts=18
  local delay_seconds=10
  local attempt

  for attempt in $(seq 1 "$max_attempts"); do
    if "$@"; then
      return 0
    fi

    if [[ "$attempt" -lt "$max_attempts" ]]; then
      printf 'IAM todavía no completó "%s" (intento %s/%s). Reintentando en %s s...\n' \
        "$description" "$attempt" "$max_attempts" "$delay_seconds" >&2
      sleep "$delay_seconds"
    fi
  done

  fail "No fue posible completar la operación IAM: ${description}"
}

section "1. Validaciones locales"

for cmd in gcloud docker curl git; do
  require_command "$cmd"
done

[[ -f "Dockerfile" ]] || fail "Debe ejecutar el script desde la raíz del repositorio: no se encontró Dockerfile."
[[ -d "app" ]] || fail "Debe ejecutar el script desde la raíz del repositorio: no se encontró app/."

git rev-parse --is-inside-work-tree >/dev/null 2>&1 \
  || fail "El directorio actual no pertenece a un repositorio Git."

ACTIVE_ACCOUNT="$(gcloud auth list --filter=status:ACTIVE --format='value(account)' | head -n1 || true)"
[[ -n "$ACTIVE_ACCOUNT" ]] || fail "No existe una cuenta activa en gcloud. Ejecute: gcloud auth login"

docker info >/dev/null 2>&1 \
  || fail "Docker no está disponible. Verifique que el daemon esté en ejecución."

printf 'Cuenta activa de gcloud: %s\n' "$ACTIVE_ACCOUNT"
printf 'Proyecto solicitado:       %s\n' "$PROJECT_ID"
printf 'Región:                    %s\n' "$REGION"
printf 'Repositorio GitHub:        %s\n' "$GITHUB_REPO"

section "2. Proyecto y facturación"

gcloud projects describe "$PROJECT_ID" >/dev/null 2>&1 \
  || fail "El proyecto ${PROJECT_ID} no existe o la cuenta activa no puede acceder a él."

gcloud config set project "$PROJECT_ID" >/dev/null

PROJECT_NUMBER="$(
  gcloud projects describe "$PROJECT_ID" \
    --format='value(projectNumber)'
)"

[[ -n "$PROJECT_NUMBER" ]] || fail "No fue posible obtener el número del proyecto."

BILLING_ENABLED="$(
  gcloud billing projects describe "$PROJECT_ID" \
    --format='value(billingEnabled)' 2>/dev/null || true
)"

if [[ "$BILLING_ENABLED" != "True" && "$BILLING_ENABLED" != "true" ]]; then
  fail "El proyecto no tiene una cuenta de facturación activa."
fi

printf 'Número del proyecto: %s\n' "$PROJECT_NUMBER"
printf 'Facturación:         habilitada\n'

section "3. Habilitación de APIs"

printf 'Habilitando APIs requeridas. Esta operación puede tardar varios minutos...\n'

gcloud services enable \
  artifactregistry.googleapis.com \
  run.googleapis.com \
  firestore.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  cloudresourcemanager.googleapis.com \
  serviceusage.googleapis.com \
  --project="$PROJECT_ID"

section "4. Artifact Registry"

if gcloud artifacts repositories describe "$AR_REPOSITORY" \
    --location="$REGION" \
    --project="$PROJECT_ID" >/dev/null 2>&1; then
  printf 'El repositorio %s ya existe. Se reutilizará.\n' "$AR_REPOSITORY"
else
  gcloud artifacts repositories create "$AR_REPOSITORY" \
    --repository-format=docker \
    --location="$REGION" \
    --description="Imágenes del Laboratorio 3 de DevOps" \
    --disable-vulnerability-scanning \
    --project="$PROJECT_ID"
fi

CLEANUP_FILE="$(mktemp)"
trap 'rm -f "$CLEANUP_FILE"' EXIT

cat > "$CLEANUP_FILE" <<'JSON'
[
  {
    "name": "delete-old-versions",
    "action": {"type": "Delete"},
    "condition": {
      "tagState": "any",
      "olderThan": "3d"
    }
  },
  {
    "name": "keep-baseline",
    "action": {"type": "Keep"},
    "condition": {
      "tagState": "tagged",
      "tagPrefixes": ["baseline"]
    }
  },
  {
    "name": "keep-recent",
    "action": {"type": "Keep"},
    "mostRecentVersions": {
      "keepCount": 3
    }
  }
]
JSON

gcloud artifacts repositories set-cleanup-policies "$AR_REPOSITORY" \
  --location="$REGION" \
  --project="$PROJECT_ID" \
  --policy="$CLEANUP_FILE" \
  --no-dry-run \
  --quiet

printf 'Cleanup policy configurada sobre %s.\n' "$AR_REPOSITORY"

section "5. Firestore"

if gcloud firestore databases describe \
    --database="(default)" \
    --project="$PROJECT_ID" >/dev/null 2>&1; then

  FIRESTORE_LOCATION="$(
    gcloud firestore databases describe \
      --database="(default)" \
      --project="$PROJECT_ID" \
      --format='value(locationId)'
  )"

  printf 'La base (default) ya existe en %s.\n' "$FIRESTORE_LOCATION"

  if [[ -n "$FIRESTORE_LOCATION" && "$FIRESTORE_LOCATION" != "$REGION" ]]; then
    fail "La base Firestore (default) ya existe en ${FIRESTORE_LOCATION}, pero el laboratorio utiliza ${REGION}. Utilice un proyecto limpio."
  fi
else
  gcloud firestore databases create \
    --database="(default)" \
    --location="$REGION" \
    --edition=standard \
    --type=firestore-native \
    --project="$PROJECT_ID"
fi

section "6. Service accounts e IAM"

ensure_service_account() {
  local name="$1"
  local email="$2"
  local display_name="$3"

  if gcloud iam service-accounts describe "$email" \
      --project="$PROJECT_ID" >/dev/null 2>&1; then
    printf 'Service account existente: %s\n' "$email"
  else
    gcloud iam service-accounts create "$name" \
      --display-name="$display_name" \
      --project="$PROJECT_ID"
  fi
}

ensure_service_account "$PUBLISHER_SA_NAME" "$PUBLISHER_SA" "GitHub Actions - publicación de imágenes"
ensure_service_account "$DEPLOYER_SA_NAME" "$DEPLOYER_SA" "GitHub Actions - despliegue en Cloud Run"
ensure_service_account "$RUNTIME_SA_NAME" "$RUNTIME_SA" "Incident API - identidad de ejecución"

printf 'Configurando permisos. Las identidades recién creadas pueden requerir tiempo de propagación...\n'

retry_iam "runtime -> Datastore User" \
  gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:${RUNTIME_SA}" \
    --role="roles/datastore.user" \
    --condition=None \
    --quiet

retry_iam "publisher -> Artifact Registry Writer" \
  gcloud artifacts repositories add-iam-policy-binding "$AR_REPOSITORY" \
    --location="$REGION" \
    --project="$PROJECT_ID" \
    --member="serviceAccount:${PUBLISHER_SA}" \
    --role="roles/artifactregistry.writer" \
    --condition=None \
    --quiet

retry_iam "deployer -> Artifact Registry Reader" \
  gcloud artifacts repositories add-iam-policy-binding "$AR_REPOSITORY" \
    --location="$REGION" \
    --project="$PROJECT_ID" \
    --member="serviceAccount:${DEPLOYER_SA}" \
    --role="roles/artifactregistry.reader" \
    --condition=None \
    --quiet

retry_iam "deployer -> Service Account User sobre runtime" \
  gcloud iam service-accounts add-iam-policy-binding "$RUNTIME_SA" \
    --project="$PROJECT_ID" \
    --member="serviceAccount:${DEPLOYER_SA}" \
    --role="roles/iam.serviceAccountUser" \
    --condition=None \
    --quiet

section "7. Workload Identity Federation"

printf 'Configurando la federación de identidad para GitHub Actions...\n'

if gcloud iam workload-identity-pools describe "$POOL_ID" \
    --location="global" \
    --project="$PROJECT_ID" >/dev/null 2>&1; then
  printf 'Workload Identity Pool existente: %s\n' "$POOL_ID"
else
  gcloud iam workload-identity-pools create "$POOL_ID" \
    --location="global" \
    --display-name="GitHub Actions - Laboratorio 3" \
    --description="Federación OIDC para el repositorio ${GITHUB_REPO}" \
    --project="$PROJECT_ID"
fi

EXPECTED_CONDITION="assertion.repository=='${GITHUB_REPO}' && assertion.ref=='refs/heads/main'"

if gcloud iam workload-identity-pools providers describe "$PROVIDER_ID" \
    --workload-identity-pool="$POOL_ID" \
    --location="global" \
    --project="$PROJECT_ID" >/dev/null 2>&1; then

  CURRENT_CONDITION="$(
    gcloud iam workload-identity-pools providers describe "$PROVIDER_ID" \
      --workload-identity-pool="$POOL_ID" \
      --location="global" \
      --project="$PROJECT_ID" \
      --format='value(attributeCondition)'
  )"

  [[ "$CURRENT_CONDITION" == "$EXPECTED_CONDITION" ]] \
    || fail "El provider ${PROVIDER_ID} ya existe con una condición distinta: ${CURRENT_CONDITION}"

  printf 'Workload Identity Provider existente y compatible: %s\n' "$PROVIDER_ID"
else
  gcloud iam workload-identity-pools providers create-oidc "$PROVIDER_ID" \
    --workload-identity-pool="$POOL_ID" \
    --location="global" \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
    --attribute-condition="$EXPECTED_CONDITION" \
    --project="$PROJECT_ID"
fi

WIF_PRINCIPAL_SET="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/attribute.repository/${GITHUB_REPO}"

retry_iam "GitHub -> impersonación de publisher" \
  gcloud iam service-accounts add-iam-policy-binding "$PUBLISHER_SA" \
    --project="$PROJECT_ID" \
    --member="$WIF_PRINCIPAL_SET" \
    --role="roles/iam.workloadIdentityUser" \
    --condition=None \
    --quiet

retry_iam "GitHub -> impersonación de deployer" \
  gcloud iam service-accounts add-iam-policy-binding "$DEPLOYER_SA" \
    --project="$PROJECT_ID" \
    --member="$WIF_PRINCIPAL_SET" \
    --role="roles/iam.workloadIdentityUser" \
    --condition=None \
    --quiet

WIF_PROVIDER="projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/providers/${PROVIDER_ID}"

section "8. Revisión baseline de Cloud Run"

printf 'Preparando la revisión baseline. La construcción y el despliegue pueden tardar varios minutos...\n'

SERVICE_EXISTS="false"
if gcloud run services describe "$RUN_SERVICE" \
    --region="$REGION" \
    --project="$PROJECT_ID" >/dev/null 2>&1; then
  SERVICE_EXISTS="true"
fi

if [[ "$SERVICE_EXISTS" == "true" ]]; then
  if gcloud run revisions describe "$BASELINE_REVISION" \
      --region="$REGION" \
      --project="$PROJECT_ID" >/dev/null 2>&1; then
    printf 'El servicio %s y la revisión %s ya existen. No se reconstruirá la baseline.\n' \
      "$RUN_SERVICE" "$BASELINE_REVISION"
  else
    fail "Ya existe un servicio Cloud Run llamado ${RUN_SERVICE}, pero no contiene la revisión ${BASELINE_REVISION}. Utilice un proyecto limpio."
  fi
else
  section "8.1 Construcción y publicación de baseline"

  gcloud auth configure-docker "${REGION}-docker.pkg.dev" --quiet

  docker build \
    --label "edu.esen.devops.release=baseline" \
    --tag "$BASELINE_IMAGE" \
    .

  docker push "$BASELINE_IMAGE"

  section "8.2 Despliegue de baseline"

  gcloud run deploy "$RUN_SERVICE" \
    --image="$BASELINE_IMAGE" \
    --region="$REGION" \
    --platform=managed \
    --revision-suffix="baseline" \
    --service-account="$RUNTIME_SA" \
    --set-env-vars="STORAGE_BACKEND=firestore,GOOGLE_CLOUD_PROJECT=${PROJECT_ID}" \
    --min=0 \
    --max=2 \
    --allow-unauthenticated \
    --project="$PROJECT_ID" \
    --quiet
fi

retry_iam "deployer -> Cloud Run Developer sobre incident-api" \
  gcloud run services add-iam-policy-binding "$RUN_SERVICE" \
    --region="$REGION" \
    --project="$PROJECT_ID" \
    --member="serviceAccount:${DEPLOYER_SA}" \
    --role="roles/run.developer" \
    --condition=None \
    --quiet

section "9. Validación funcional"

SERVICE_URL="$(
  gcloud run services describe "$RUN_SERVICE" \
    --region="$REGION" \
    --project="$PROJECT_ID" \
    --format='value(status.url)'
)"

LATEST_READY_REVISION="$(
  gcloud run services describe "$RUN_SERVICE" \
    --region="$REGION" \
    --project="$PROJECT_ID" \
    --format='value(status.latestReadyRevisionName)'
)"

[[ -n "$SERVICE_URL" ]] || fail "No fue posible obtener la URL del servicio Cloud Run."

printf 'URL del servicio:          %s\n' "$SERVICE_URL"
printf 'Última revisión preparada: %s\n' "$LATEST_READY_REVISION"

HEALTH_OK="false"
for attempt in $(seq 1 10); do
  if curl --fail --silent --show-error "${SERVICE_URL}/health" >/dev/null 2>&1; then
    HEALTH_OK="true"
    break
  fi
  printf 'Intento %s/10 para /health. Reintentando en 10 s...\n' "$attempt"
  sleep 10
done

[[ "$HEALTH_OK" == "true" ]] || fail "El endpoint /health no respondió satisfactoriamente."
printf 'GET /health: OK\n'

INCIDENTS_OK="false"
INCIDENTS_BODY=""

for attempt in $(seq 1 24); do
  RESPONSE_FILE="$(mktemp)"

  if curl --fail --silent --show-error \
      "${SERVICE_URL}/incidents" \
      --output "$RESPONSE_FILE" 2>/dev/null; then
    INCIDENTS_BODY="$(cat "$RESPONSE_FILE")"
    rm -f "$RESPONSE_FILE"
    INCIDENTS_OK="true"
    break
  fi

  rm -f "$RESPONSE_FILE"
  printf 'Intento %s/24 para /incidents. Reintentando en 15 s...\n' "$attempt"
  sleep 15
done

[[ "$INCIDENTS_OK" == "true" ]] \
  || fail "El endpoint /incidents no pudo acceder correctamente a Firestore."

printf 'GET /incidents: OK\n'
printf 'Respuesta: %s\n' "$INCIDENTS_BODY"

section "ENTORNO DEL LABORATORIO 3 PREPARADO"

cat <<EOF
Proyecto:
  ${PROJECT_ID}

Número del proyecto:
  ${PROJECT_NUMBER}

Región:
  ${REGION}

Artifact Registry:
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPOSITORY}

Imagen baseline:
  ${BASELINE_IMAGE}

Firestore:
  (default)

Cloud Run:
  Servicio: ${RUN_SERVICE}
  URL: ${SERVICE_URL}
  Revisión baseline: ${BASELINE_REVISION}
  Última revisión lista: ${LATEST_READY_REVISION}

Service accounts:
  Publisher: ${PUBLISHER_SA}
  Deployer:  ${DEPLOYER_SA}
  Runtime:   ${RUNTIME_SA}

Workload Identity Provider:
  ${WIF_PROVIDER}

Repositorio autorizado:
  ${GITHUB_REPO}

Referencia autorizada:
  refs/heads/main

------------------------------------------------------------
Variables de repositorio para GitHub Actions
------------------------------------------------------------

GCP_PROJECT_ID=${PROJECT_ID}
GCP_REGION=${REGION}
ARTIFACT_REPOSITORY=${AR_REPOSITORY}
IMAGE_NAME=${IMAGE_NAME}
CLOUD_RUN_SERVICE=${RUN_SERVICE}
WIF_PROVIDER=${WIF_PROVIDER}
PUBLISHER_SERVICE_ACCOUNT=${PUBLISHER_SA}
DEPLOYER_SERVICE_ACCOUNT=${DEPLOYER_SA}
RUNTIME_SERVICE_ACCOUNT=${RUNTIME_SA}

Estas variables NO son llaves privadas y no deben reemplazarse por una
service account key JSON.

Preparación finalizada correctamente.
EOF
```

### 7.3 Ejecutar el bootstrap

Dar permiso de ejecución:

```bash
chmod +x bootstrap_lab3.sh
```

Ejecutar:

```bash
./bootstrap_lab3.sh
```

La operación puede tardar varios minutos. No debe interrumpirse únicamente porque la habilitación de APIs, la construcción de la imagen o el despliegue tarden algunos minutos.

Si finaliza correctamente debe aparecer:

```text
============================================================
ENTORNO DEL LABORATORIO 3 PREPARADO
============================================================
```

La salida final mostrará, entre otros datos:

- proyecto y número del proyecto;
- ubicación de Artifact Registry;
- imagen `baseline`;
- URL del servicio Cloud Run;
- revisión `incident-api-baseline`;
- tres service accounts;
- Workload Identity Provider;
- variables que deben configurarse en GitHub.

### 7.4 Verificaciones posteriores

Ejecutar:

```bash
gcloud iam service-accounts list \
  --project="$PROJECT_ID"
```

Deben aparecer, como mínimo:

```text
gha-publisher@...
gha-deployer@...
incident-api-runtime@...
```

Comprobar Firestore:

```bash
gcloud firestore databases describe \
  --database="(default)" \
  --project="$PROJECT_ID"
```

Comprobar las políticas de limpieza:

```bash
gcloud artifacts repositories list-cleanup-policies devops-images \
  --location="$REGION" \
  --project="$PROJECT_ID"
```

Comprobar las revisiones de Cloud Run:

```bash
gcloud run revisions list \
  --service=incident-api \
  --region="$REGION" \
  --project="$PROJECT_ID"
```

En este momento debe existir:

```text
incident-api-baseline
```

Comprobar el tráfico:

```bash
gcloud run services describe incident-api \
  --region="$REGION" \
  --project="$PROJECT_ID" \
  --format="yaml(status.traffic)"
```

Antes de desarrollar la entrega continua, `incident-api-baseline` debe recibir el 100 % del tráfico.

## 8. Variables del repositorio en GitHub

El bootstrap imprime ocho valores al finalizar. Deben crearse en:

```text
GitHub
→ Settings
→ Secrets and variables
→ Actions
→ Variables
→ New repository variable
```

Crear exactamente estas variables:

```text
GCP_PROJECT_ID
GCP_REGION
ARTIFACT_REPOSITORY
IMAGE_NAME
CLOUD_RUN_SERVICE
WIF_PROVIDER
PUBLISHER_SERVICE_ACCOUNT
DEPLOYER_SERVICE_ACCOUNT
RUNTIME_SERVICE_ACCOUNT
```

Ejemplo de formato:

```text
GCP_PROJECT_ID=devops-pareja-07
GCP_REGION=us-central1
ARTIFACT_REPOSITORY=devops-images
IMAGE_NAME=incident-api
CLOUD_RUN_SERVICE=incident-api
WIF_PROVIDER=projects/123456789/locations/global/workloadIdentityPools/github-lab3/providers/github
PUBLISHER_SERVICE_ACCOUNT=gha-publisher@devops-pareja-07.iam.gserviceaccount.com
DEPLOYER_SERVICE_ACCOUNT=gha-deployer@devops-pareja-07.iam.gserviceaccount.com
RUNTIME_SERVICE_ACCOUNT=incident-api-runtime@devops-pareja-07.iam.gserviceaccount.com
```

Estos valores son configuración, no llaves privadas. **No se debe crear un secret llamado `GCP_KEY`, `GOOGLE_CREDENTIALS`, `SERVICE_ACCOUNT_JSON` ni equivalente.**

Añadir también esta línea a `.gitignore` y `.dockerignore`:

```text
gha-creds-*.json
```

La acción de autenticación de Google puede crear archivos temporales de credenciales dentro del workspace. No deben incorporarse al repositorio ni al contexto de construcción.

## 9. Extensión del pipeline de integración continua

El workflow desarrollado en el Laboratorio 2 se conserva como punto de partida. No se debe eliminar la validación existente para sustituirla por un workflow completamente distinto.

La estructura conceptual esperada es:

```text
Lint ─────────┐
              ├──> Container ───> Deploy candidate
Test ─────────┘
```

`Deploy candidate` solo debe poder ejecutarse en un `push` a `main`.

### 9.1 Comportamiento en pull requests

Para:

```yaml
pull_request:
  branches:
    - main
```

el pipeline debe:

- ejecutar `Lint`;
- ejecutar `Test`;
- ejecutar `Container`;
- construir correctamente la imagen;
- no autenticarse en Google Cloud;
- no publicar imágenes;
- no crear revisiones de Cloud Run;
- no modificar tráfico.

### 9.2 Comportamiento después de integrar a `main`

Para:

```yaml
push:
  branches:
    - main
```

el pipeline debe ejecutar nuevamente las validaciones y, solo si son satisfactorias:

1. construir la imagen;
2. etiquetarla con el SHA completo del commit;
3. autenticarse mediante OIDC/WIF utilizando `gha-publisher`;
4. publicar la imagen en Artifact Registry;
5. obtener su digest;
6. pasar ese digest al job de despliegue;
7. autenticarse mediante OIDC/WIF utilizando `gha-deployer`;
8. desplegar **la imagen por digest** como una nueva revisión de `incident-api`;
9. asignarle el traffic tag `candidate`;
10. crearla con `0 %` del tráfico principal;
11. verificar `/health`;
12. verificar `/incidents`.

### 9.3 Identidad de la imagen

Construir la referencia a la imagen a partir de las variables del repositorio:

```text
REGION-docker.pkg.dev/PROJECT_ID/ARTIFACT_REPOSITORY/IMAGE_NAME
```

El tag publicado debe ser:

```text
${GITHUB_SHA}
```

Por ejemplo:

```text
us-central1-docker.pkg.dev/devops-pareja-07/devops-images/incident-api:4af7...
```

No se permite utilizar únicamente:

```text
latest
```

como identidad del artefacto evaluado.

Una vez publicada, debe recuperarse el digest:

```text
sha256:...
```

El job de despliegue debe recibir o recuperar este valor y formar una referencia equivalente a:

```text
us-central1-docker.pkg.dev/.../incident-api@sha256:...
```

La revisión de Cloud Run debe desplegar esa referencia.

### 9.4 Autenticación desde GitHub Actions

Los jobs que utilizan Google Cloud necesitan:

```yaml
permissions:
  contents: read
  id-token: write
```

La autenticación debe realizarse mediante:

```yaml
uses: google-github-actions/auth@v3
```

y las variables:

```text
WIF_PROVIDER
PUBLISHER_SERVICE_ACCOUNT
DEPLOYER_SERVICE_ACCOUNT
```

Para publicar se utiliza `PUBLISHER_SERVICE_ACCOUNT`.

Para crear la revisión candidata y modificar el servicio se utiliza `DEPLOYER_SERVICE_ACCOUNT`.

No se debe utilizar una llave JSON.

Después de `auth`, puede utilizarse:

```yaml
uses: google-github-actions/setup-gcloud@v3
```

cuando el job requiera comandos `gcloud`.

### 9.5 Publicación en Artifact Registry

La publicación debe ocurrir únicamente después de que `Lint` y `Test` hayan finalizado correctamente.

El job deberá autenticar Docker contra:

```text
${GCP_REGION}-docker.pkg.dev
```

y publicar la imagen etiquetada con el commit SHA.

Al finalizar debe ser posible localizar el artefacto con:

```bash
gcloud artifacts docker images list \
  "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${ARTIFACT_REPOSITORY}/${IMAGE_NAME}" \
  --include-tags
```

Para una ejecución concreta debe poder explicarse:

```text
commit SHA
    ↓
tag de Artifact Registry
    ↓
digest sha256
```

## 10. Despliegue de la revisión candidata

Después de publicar la imagen se debe crear una nueva revisión del **mismo servicio** `incident-api`.

El despliegue debe conservar:

```text
STORAGE_BACKEND=firestore
GOOGLE_CLOUD_PROJECT=<proyecto>
service account de runtime=incident-api-runtime
```

La revisión candidata debe utilizar:

```text
--no-traffic
--tag=candidate
```

No se debe crear un segundo servicio Cloud Run.

La lógica equivalente es:

```bash
gcloud run deploy incident-api \
  --image="IMAGEN_POR_DIGEST" \
  --region="$GCP_REGION" \
  --service-account="$RUNTIME_SERVICE_ACCOUNT" \
  --set-env-vars="STORAGE_BACKEND=firestore,GOOGLE_CLOUD_PROJECT=${GCP_PROJECT_ID}" \
  --no-traffic \
  --tag=candidate
```

El nombre exacto de la revisión debe quedar asociado a la ejecución. Se recomienda utilizar como `revision-suffix` una forma corta del commit SHA para facilitar la trazabilidad.

### 10.1 Obtener la URL candidata

El traffic tag produce una URL que permite acceder directamente a esa revisión incluso cuando recibe 0 % del tráfico principal.

El workflow debe obtener la URL etiquetada. Una forma válida consiste en consultar el servicio en JSON y seleccionar la entrada cuyo `tag` sea `candidate`:

```bash
CANDIDATE_URL="$(
  gcloud run services describe "$CLOUD_RUN_SERVICE" \
    --region="$GCP_REGION" \
    --project="$GCP_PROJECT_ID" \
    --format=json |
  jq -r '.status.traffic[] | select(.tag == "candidate") | .url'
)"
```

El job debe fallar si `CANDIDATE_URL` queda vacío o si vale `null`.

### 10.2 Verificar la revisión candidata

Antes de considerar correcto el despliegue:

```bash
curl --fail --silent --show-error "${CANDIDATE_URL}/health"
```

y:

```bash
curl --fail --silent --show-error "${CANDIDATE_URL}/incidents"
```

deben finalizar correctamente.

`/health` comprueba que la aplicación responde. `/incidents` obliga a la aplicación desplegada a utilizar su repositorio de datos y, por tanto, también comprueba que la identidad `incident-api-runtime` puede acceder a Firestore.

Después de este job, comprobar:

```bash
gcloud run services describe incident-api \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --format="yaml(status.traffic)"
```

Debe existir una revisión etiquetada como `candidate`, pero la revisión que servía tráfico antes del despliegue debe conservar el 100 % del tráfico principal.

## 11. Promoción manual

Crear un workflow independiente:

```text
.github/workflows/promote.yml
```

El workflow debe utilizar:

```yaml
on:
  workflow_dispatch:
```

La promoción no debe ocurrir automáticamente por un `push`.

### 11.1 Secuencia requerida

El workflow de promoción debe:

1. autenticarse con `DEPLOYER_SERVICE_ACCOUNT`;
2. obtener la revisión actualmente asociada al tag `candidate`;
3. obtener la URL candidata;
4. ejecutar nuevamente:
   - `GET /health`;
   - `GET /incidents`;
5. finalizar con error si cualquiera de las verificaciones falla;
6. mover el 100 % del tráfico hacia **esa revisión concreta**;
7. verificar el servicio mediante su URL principal.

Para obtener la revisión asociada al tag:

```bash
CANDIDATE_REVISION="$(
  gcloud run services describe "$CLOUD_RUN_SERVICE" \
    --region="$GCP_REGION" \
    --project="$GCP_PROJECT_ID" \
    --format=json |
  jq -r '.status.traffic[] | select(.tag == "candidate") | .revisionName'
)"
```

El workflow debe comprobar que el valor no esté vacío ni sea `null`.

La promoción debe utilizar la revisión resuelta, por ejemplo:

```bash
gcloud run services update-traffic "$CLOUD_RUN_SERVICE" \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --to-revisions="${CANDIDATE_REVISION}=100"
```

Después:

```bash
gcloud run services describe "$CLOUD_RUN_SERVICE" \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --format="yaml(status.traffic)"
```

debe mostrar la revisión promovida con el 100 % del tráfico.

## 12. Rollback manual

Crear un workflow independiente:

```text
.github/workflows/rollback.yml
```

También debe utilizar:

```yaml
on:
  workflow_dispatch:
```

El workflow debe recibir como entrada el nombre de una revisión existente que se utilizará como destino del rollback.

Una estructura válida para la entrada es:

```yaml
on:
  workflow_dispatch:
    inputs:
      revision:
        description: "Revisión de Cloud Run a restaurar"
        required: true
        type: string
```

### 12.1 Secuencia requerida

El workflow debe:

1. autenticarse con `DEPLOYER_SERVICE_ACCOUNT`;
2. comprobar que la revisión indicada existe;
3. comprobar que pertenece al servicio `incident-api`;
4. mover el 100 % del tráfico hacia esa revisión;
5. verificar `/health`;
6. verificar `/incidents`.

El cambio de tráfico debe tener la forma:

```bash
gcloud run services update-traffic "$CLOUD_RUN_SERVICE" \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --to-revisions="${TARGET_REVISION}=100"
```

No se permite:

```text
docker build
docker push
gcloud run deploy ...
```

dentro del workflow de rollback.

Un rollback de aplicación restaura una revisión existente. No reconstruye el código histórico.

## 13. Prueba completa de entrega y recuperación

Cuando los workflows estén terminados, ejecutar la siguiente secuencia.

### Paso 1. Generar una nueva entrega

Realizar un cambio válido y pequeño en una rama distinta de `main`.

Ejemplo de flujo Git:

```bash
git switch -c lab3/cd
```

Realizar los cambios de workflows y documentación:

```bash
git add .
git commit -m "Implement continuous delivery"
git push -u origin lab3/cd
```

Crear un pull request hacia `main`.

Antes del merge deben completarse los checks requeridos por el ruleset.

### Paso 2. Integrar a `main`

Hacer merge del pull request.

El `push` resultante sobre `main` debe:

- ejecutar CI;
- publicar una imagen;
- crear una revisión candidata;
- verificarla;
- mantener la revisión anterior sirviendo el 100 % del tráfico.

### Paso 3. Registrar identidades

En `EVIDENCIAS.md` registrar:

```text
Commit:
Tag:
Digest:
Revisión candidata:
URL candidata:
```

Debe ser posible demostrar que el digest desplegado corresponde a la imagen publicada para ese commit.

### Paso 4. Promover

Ejecutar manualmente:

```text
Actions
→ workflow de promoción
→ Run workflow
```

Comprobar que la candidata pasa al 100 %.

### Paso 5. Crear estado persistente

Obtener la URL principal:

```bash
SERVICE_URL="$(
  gcloud run services describe incident-api \
    --region="$REGION" \
    --project="$PROJECT_ID" \
    --format='value(status.url)'
)"
```

Crear un incidente:

```bash
curl --fail --silent --show-error \
  -X POST "${SERVICE_URL}/incidents" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Validación de rollback",
    "description": "Incidente creado antes de restaurar la revisión anterior",
    "priority": "low"
  }'
```

Comprobar:

```bash
curl --fail --silent --show-error \
  "${SERVICE_URL}/incidents"
```

### Paso 6. Ejecutar rollback

Localizar las revisiones:

```bash
gcloud run revisions list \
  --service=incident-api \
  --region="$REGION" \
  --project="$PROJECT_ID"
```

Ejecutar manualmente `rollback.yml` indicando:

```text
incident-api-baseline
```

como revisión de destino para esta demostración.

### Paso 7. Comprobar rollback

Comprobar el tráfico:

```bash
gcloud run services describe incident-api \
  --region="$REGION" \
  --project="$PROJECT_ID" \
  --format="yaml(status.traffic)"
```

`incident-api-baseline` debe recibir el 100 %.

Volver a consultar:

```bash
curl --fail --silent --show-error \
  "${SERVICE_URL}/incidents"
```

El incidente creado antes del rollback debe continuar almacenado.

Esta comprobación muestra una propiedad importante: **restaurar una revisión de la aplicación no revierte automáticamente el estado persistente de Firestore**.

## 14. Evidencias

Se continuará utilizando el archivo:

```text
EVIDENCIAS.md
```

del repositorio. Añadir una sección:

```markdown
# Laboratorio 3. Entrega continua y despliegue en la nube
```

Debe incluir, como mínimo:

### 14.1 Preparación del entorno

- ID del proyecto GCP.
- captura del presupuesto de USD 1;
- salida resumida que muestre las tres service accounts;
- salida de la cleanup policy;
- revisión `incident-api-baseline`;
- comprobación inicial de `/health` y `/incidents`.

### 14.2 Publicación

- enlace a la ejecución de GitHub Actions;
- commit SHA;
- tag utilizado en Artifact Registry;
- digest de la imagen;
- evidencia de que no se almacenó una llave JSON.

### 14.3 Revisión candidata

- nombre de la revisión;
- URL asociada al tag `candidate`;
- resultado de `GET /health`;
- resultado de `GET /incidents`;
- evidencia de que la candidata tenía 0 % del tráfico principal antes de la promoción.

### 14.4 Promoción

- enlace a la ejecución manual de `promote.yml`;
- revisión promovida;
- tráfico después de la promoción.

### 14.5 Rollback

- enlace a la ejecución manual de `rollback.yml`;
- revisión restaurada;
- tráfico después del rollback;
- evidencia de que el workflow no reconstruyó ni publicó una imagen;
- resultado de `/health` y `/incidents` después del rollback;
- evidencia de que el incidente persistente continuó existiendo.

### 14.6 Explicación técnica

Responder brevemente:

1. ¿Por qué se publica la imagen con el SHA del commit y se despliega mediante digest?
2. ¿Qué diferencia existe entre `gha-publisher`, `gha-deployer` e `incident-api-runtime`?
3. ¿Por qué una revisión candidata puede probarse aunque reciba 0 % del tráfico principal?
4. ¿Por qué el rollback no debe reconstruir una versión anterior?
5. ¿Por qué el incidente almacenado en Firestore continúa existiendo después del rollback de la aplicación?

## 15. Entrega

La entrega consiste en el estado del mismo repositorio utilizado desde el Laboratorio 2.

Como mínimo, el repositorio deberá contener:

```text
.github/
└── workflows/
    ├── ci.yml
    ├── promote.yml
    └── rollback.yml

app/
Dockerfile
.dockerignore
.gitignore
EVIDENCIAS.md
bootstrap_lab3.sh
```

El nombre exacto del workflow de CI puede conservar el utilizado en el Laboratorio 2. No es obligatorio dividir la solución en exactamente tres archivos si otra organización satisface todos los requisitos, pero promoción y rollback deben continuar siendo acciones manuales claramente identificables.

El código debe integrarse a `main` mediante pull request.

No deben eliminarse los recursos de Google Cloud antes de que el laboratorio haya sido revisado.

## 16. Evaluación

| Criterio | Puntos |
| --- | ---: |
| Preparación y control del entorno de Google Cloud | 5 |
| Autenticación OIDC + WIF sin credenciales permanentes | 15 |
| Separación de identidades y mínimo privilegio | 10 |
| Publicación, versionado y trazabilidad del artefacto | 15 |
| Despliegue de la misma imagen en Cloud Run | 15 |
| Revisión candidata y verificación previa | 10 |
| Control explícito de promoción | 10 |
| Rollback a revisión anterior sin reconstrucción | 15 |
| Evidencia técnica y razonamiento | 5 |
| **Total** | **100** |

### 16.1 Preparación y control del entorno, 5 puntos

Se verifica que el proyecto, presupuesto, Artifact Registry, cleanup policy, Firestore, Cloud Run y las identidades requeridas hayan quedado correctamente preparados.

### 16.2 OIDC + WIF, 15 puntos

Se verifica que GitHub Actions obtenga credenciales temporales mediante OIDC y Workload Identity Federation. La presencia de una llave JSON permanente invalida este criterio.

### 16.3 Separación de identidades, 10 puntos

Se verifica que publicación, despliegue y ejecución utilicen identidades diferentes y que los permisos correspondan a sus responsabilidades.

### 16.4 Publicación y trazabilidad, 15 puntos

Se debe demostrar la relación:

```text
commit → tag → digest
```

### 16.5 Despliegue del mismo artefacto, 15 puntos

La revisión candidata debe utilizar el digest del artefacto ya publicado. No se debe reconstruir la imagen entre publicación y despliegue.

### 16.6 Revisión candidata, 10 puntos

La revisión debe desplegarse sin tráfico principal, ser accesible por el tag `candidate` y superar las verificaciones de la API.

### 16.7 Promoción, 10 puntos

La revisión candidata solo debe recibir el 100 % del tráfico después de una ejecución manual del workflow de promoción.

### 16.8 Rollback, 15 puntos

Debe restaurarse una revisión anterior existente mediante cambio de tráfico y sin reconstrucción.

### 16.9 Evidencia y razonamiento, 5 puntos

Las evidencias deben permitir reconstruir lo ocurrido y las respuestas deben mostrar comprensión de las decisiones implementadas.

## 17. Errores frecuentes

### El bootstrap indica que una service account no existe inmediatamente después de crearla

La versión suministrada incorpora reintentos automáticos para operaciones IAM. No se debe crear una segunda service account con otro nombre.

### `PERMISSION_DENIED` al autenticar desde GitHub Actions

Comprobar:

- que el workflow tiene `id-token: write`;
- que `WIF_PROVIDER` contiene el nombre completo del provider;
- que se usa el número del proyecto dentro del recurso del provider;
- que el repositorio corresponde exactamente al configurado durante el bootstrap;
- que la ejecución procede de `refs/heads/main` cuando intenta publicar o desplegar;
- que se utiliza la service account correcta para la operación.

### Un pull request intenta publicar o desplegar

La condición del job o de los steps es incorrecta. Los cambios de infraestructura del laboratorio deben ejecutarse únicamente en `push` a `main`.

### La revisión candidata recibe tráfico inmediatamente

El despliegue debe incluir `--no-traffic`.

### No aparece una URL para `candidate`

Comprobar que el despliegue incluyó:

```text
--tag=candidate
```

y consultar:

```bash
gcloud run services describe incident-api \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --format="yaml(status.traffic)"
```

### `/health` funciona pero `/incidents` falla

La aplicación inició correctamente, pero existe un problema en la ruta hacia Firestore. Revisar:

- `STORAGE_BACKEND=firestore`;
- `GOOGLE_CLOUD_PROJECT`;
- service account asignada a la revisión;
- permiso `roles/datastore.user` de `incident-api-runtime`;
- existencia de Firestore `(default)`.

### El rollback crea otra revisión

El workflow está desplegando en lugar de modificar tráfico. El rollback de este laboratorio debe utilizar `gcloud run services update-traffic`.

## 18. Referencias técnicas

- GitHub Actions, autenticación con Google Cloud: <https://github.com/google-github-actions/auth>
- GitHub Action para `gcloud`: <https://github.com/google-github-actions/setup-gcloud>
- Workload Identity Federation para pipelines: <https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines>
- Artifact Registry, autenticación de Docker: <https://cloud.google.com/artifact-registry/docs/docker/authentication>
- Artifact Registry, cleanup policies: <https://cloud.google.com/artifact-registry/docs/repositories/cleanup-policy>
- Firestore, administración de bases de datos: <https://cloud.google.com/firestore/docs/manage-databases>
- Cloud Run, despliegue: <https://cloud.google.com/run/docs/deploying>
- Cloud Run, revisiones, tráfico y rollback: <https://cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration>

## 19. Nota sobre el provider de GitHub

Para mantener la preparación operable dentro del alcance del laboratorio, el provider restringe el acceso por nombre completo de repositorio y por `refs/heads/main`. En sistemas de producción debe considerarse también el uso de identificadores numéricos estables de GitHub para reducir riesgos asociados al cambio o reutilización de nombres.
