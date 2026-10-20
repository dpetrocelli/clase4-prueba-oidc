# clase4-prueba-oidc

Prueba de las identidades de la clase 4 por OIDC (sin claves de GCP guardadas):
`infra-deployer` (VMs y estado) y `app-deployer` (push a Artifact Registry), más
el par SSH que va a usar la VM (secret `SSH_PRIVATE_KEY` + variable `SSH_PUBLIC_KEY`).

Correrlo: Actions → prueba → Run workflow, o cualquier push a `main`..
