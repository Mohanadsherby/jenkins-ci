# jenkins-ci

Jenkins pipeline definitions, kept out of the application repos so a job's
script can change without touching application history.

## Layout

```
petclinic/
├── dev/<service>_Jenkinsfile
└── stg/<service>_Jenkinsfile
```

One file per service per environment. Point each Jenkins job at
*Pipeline script from SCM* → this repo → script path
`petclinic/<env>/<service>_Jenkinsfile`.

## What a pipeline does

Every pipeline is a full build — each environment produces its own image and
pushes it to its own registry path. Nothing is retagged or copied between
environments.

1. Check out [spring-petclinic-microservices](https://github.com/Mohanadsherby/spring-petclinic-microservices) at `main`
2. Tag = `<short git SHA>-<Jenkins build number>`, e.g. `0eee00d-14`
3. `./mvnw -pl spring-petclinic-<service> -am package -DskipTests`
4. Build, Trivy-scan, and push `docker-hosted/<env>/<service>:<tag>`
5. Write that tag into `helm/petclinic/values/<env>/<service>.yaml` in the
   config repo — ArgoCD watches it and deploys

The dev and stg files are identical apart from `DEPLOY_ENV`, which selects
both the registry path and the values file.

## What the pipelines depend on

| Thing | Value |
|---|---|
| Config repo (ArgoCD watches it) | [argocd-gitops](https://github.com/Mohanadsherby/argocd-gitops) |
| Registry, from Jenkins | `localhost:8084` (Nexus) |
| Registry, from inside the cluster | `nexus:8081` (set in the values files) |
| Jenkins credential — Nexus | `nexus-credentials` |
| Jenkins credential — GitHub push | `github-credentials` |

Trivy scans HIGH/CRITICAL in both environments but does not fail the build
(`--exit-code 0 ... || true`). To make stg block on findings, drop the
`|| true` and set `--exit-code 1` in `petclinic/stg/*_Jenkinsfile`.
