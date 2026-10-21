# clase4-prueba-oidc · end state

Lo de la clase 4 terminado: una API en Docker que llega a una VM en GCP con dos pipelines y
**ninguna clave de GCP guardada** (OIDC).

| Pipeline | Cuándo corre | Qué hace | Identidad |
|---|---|---|---|
| [`infra`](.github/workflows/infra.yml) | cambia `infra/` | OpenTofu: VM Debian con Docker (por startup script), firewall 8080 y 22 | `infra-deployer` |
| [`app`](.github/workflows/app.yml) | cambia `app/` | tests → `docker build` + push a Artifact Registry → SSH a la VM, `docker pull` y `docker run` | `app-deployer` |
| [`prueba`](.github/workflows/prueba.yml) | a mano | Verifica que cada identidad pueda lo suyo y nada más, y que el par SSH coincida | las dos |

Usalo como template: **Use this template → Create a new repository**.

## 1. Lo que tiene que existir en GCP (una vez, como Owner)

Supone el bucket `<PROYECTO>-tfstate` (clase 2) y el repositorio de Artifact Registry
`infra-cloud-template` en `us-central1` (clase 3). Reemplazá `TU_PROYECTO` y `TU_USUARIO/TU_REPO`:

```bash
PROJECT_ID=TU_PROYECTO
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
REPO=TU_USUARIO/TU_REPO          # exacto, con mayúsculas
INFRA_SA=infra-deployer@$PROJECT_ID.iam.gserviceaccount.com
APP_SA=app-deployer@$PROJECT_ID.iam.gserviceaccount.com
COMPUTE_SA=$PROJECT_NUMBER-compute@developer.gserviceaccount.com

gcloud services enable iam.googleapis.com iamcredentials.googleapis.com sts.googleapis.com \
  artifactregistry.googleapis.com compute.googleapis.com

# Dos identidades de mínimo privilegio.
gcloud iam service-accounts create infra-deployer --display-name="Pipeline infra"
gcloud iam service-accounts create app-deployer --display-name="Pipeline app"

# infra-deployer: VMs, firewall, el estado de OpenTofu, y usar la cuenta de Compute para la VM.
gcloud projects add-iam-policy-binding $PROJECT_ID --member=serviceAccount:$INFRA_SA --role=roles/compute.instanceAdmin.v1
gcloud projects add-iam-policy-binding $PROJECT_ID --member=serviceAccount:$INFRA_SA --role=roles/compute.securityAdmin
gcloud storage buckets add-iam-policy-binding gs://$PROJECT_ID-tfstate --member=serviceAccount:$INFRA_SA --role=roles/storage.objectAdmin
gcloud iam service-accounts add-iam-policy-binding $COMPUTE_SA --member=serviceAccount:$INFRA_SA --role=roles/iam.serviceAccountUser

# app-deployer: escribir imágenes y ver la IP de la VM. No puede crear ni borrar VMs.
gcloud artifacts repositories add-iam-policy-binding infra-cloud-template --location=us-central1 \
  --member=serviceAccount:$APP_SA --role=roles/artifactregistry.writer
gcloud projects add-iam-policy-binding $PROJECT_ID --member=serviceAccount:$APP_SA --role=roles/compute.viewer

# La VM hace docker pull con la cuenta de Compute: que pueda leer las imágenes.
gcloud artifacts repositories add-iam-policy-binding infra-cloud-template --location=us-central1 \
  --member=serviceAccount:$COMPUTE_SA --role=roles/artifactregistry.reader

# Confiar en GitHub, pero solo en TU repo (la condición es toda la seguridad).
gcloud iam workload-identity-pools create github-pool --location=global
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository=='$REPO'"

# Que tu repo pueda "ser" cada identidad.
MEMBER="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/$REPO"
gcloud iam service-accounts add-iam-policy-binding $INFRA_SA --role=roles/iam.workloadIdentityUser --member="$MEMBER"
gcloud iam service-accounts add-iam-policy-binding $APP_SA --role=roles/iam.workloadIdentityUser --member="$MEMBER"

# El nombre completo del provider, para la variable del repo.
gcloud iam workload-identity-pools providers describe github-provider \
  --location=global --workload-identity-pool=github-pool --format="value(name)"
```

## 2. La clave SSH de la VM

```bash
ssh-keygen -t ed25519 -f ~/.ssh/clase4 -N "" -C clase4
```

`clase4.pub` va a la VM (variable, no es secreto). `clase4` (la privada) es el **único secret**:
el pipeline `app` la usa para entrar a la VM.

## 3. Variables y secret del repo

**Settings → Secrets and variables → Actions** (o con `gh`, como abajo):

| Tipo | Nombre | Valor |
|---|---|---|
| Variable | `GCP_PROJECT_ID` | tu project id |
| Variable | `GCP_WORKLOAD_IDENTITY_PROVIDER` | la salida del último comando (`projects/<NÚMERO>/...`) |
| Variable | `GCP_INFRA_SERVICE_ACCOUNT` | `infra-deployer@TU_PROYECTO.iam.gserviceaccount.com` |
| Variable | `GCP_APP_SERVICE_ACCOUNT` | `app-deployer@TU_PROYECTO.iam.gserviceaccount.com` |
| Variable | `SSH_PUBLIC_KEY` | el contenido de `~/.ssh/clase4.pub` |
| Variable | `VM_NAME` | opcional; por defecto `prueba-vm` |
| Secret | `SSH_PRIVATE_KEY` | el contenido de `~/.ssh/clase4` |

```bash
gh variable set SSH_PUBLIC_KEY -R $REPO --body "$(cat ~/.ssh/clase4.pub)"
gh secret set SSH_PRIVATE_KEY -R $REPO < ~/.ssh/clase4
```

## 4. Orden del primer deploy

1. **Actions → prueba → Run workflow**: los tres jobs en verde. Si falla con
   `getAccessToken denied`, el permiso tarda hasta 5 minutos: esperá y **Re-run failed jobs**.
2. **Actions → infra → Run workflow**: `Plan: 3 to add` y la IP en los outputs. La VM instala
   Docker en 1–2 minutos.
3. **Actions → app → Run workflow**: test, build y deploy. Abrí `http://<IP>:8080/docs`.

Después, cada push que toque `app/` despliega solo; uno que toque `infra/`, corre solo `infra`.

## 5. Al terminar: borrar la VM

```bash
cd infra
tofu init -backend-config="bucket=$PROJECT_ID-tfstate"
tofu destroy -var project_id=$PROJECT_ID -var ssh_public_key="$(cat ~/.ssh/clase4.pub)"
```
