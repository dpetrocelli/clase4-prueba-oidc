# clase4-prueba-oidc

Prueba mínima de OIDC (Workload Identity Federation) entre GitHub Actions y GCP:
sin app, sin infra, sin claves guardadas. El workflow `prueba` se loguea como la
service account `ci-prueba` (solo lectura), lista imágenes de Artifact Registry
y demuestra que no puede crear un bucket.

Correrlo: Actions → prueba → Run workflow, o cualquier push a `main`.
