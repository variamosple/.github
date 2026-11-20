# variamosple/.github

Shared GitHub Actions workflows of the VariaMos organization.

| Path | What it is |
|---|---|
| `.github/workflows/deploy.yml` | Reusable deployment workflow: build, check, publish and deploy one service |
| `.github/workflows/stryker.yml` | Reusable mutation testing (Stryker), score published as a commit status |
| `.github/workflows/npm-package-ci.yml` | Reusable checks of the npm packages (`variamos_components`, `variamos_security`) |
| `workflow-templates/deploy.yml` | Starter workflow that calls the deployment, offered in "Actions > New workflow" of every repository |

## Deploy a service

Add `.github/workflows/deploy.yml` to the service repository:

```yaml
name: deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: variamosple/.github/.github/workflows/deploy.yml@v1
    with:
      image: vms_language_reviews          # image name on Docker Hub
      service: vmslanguagereviewsaws       # service name in the production compose file
      health-url: https://app.variamos.com/vms_language_reviews/health
      health-name: vms_language_reviews
    secrets: inherit
```

The workflow runs in the context of the calling repository: it reads that
repository's secrets and variables, and its `production` environment (created
on the first run).

What it does:

1. builds the image from the repository (`context`, `dockerfile`);
2. checks it with Trivy and looks for environment or key files in it;
3. publishes it on Docker Hub with the `main` tag;
4. on the server, keeps the image in service, then pulls and restarts **only
   this service**;
5. with `health-url`: waits up to 5 minutes for the new container to answer
   `{"serviceName": <health-name>, "status": "UP"}` with a fresh uptime, and
   otherwise puts the previous image back.

### Inputs

| Input | Default | Use |
|---|---|---|
| `image` | required | Image name on Docker Hub, without registry, account or tag |
| `service` | required | Service name in the production compose file |
| `context` | `.` | Build context |
| `dockerfile` | `./Dockerfile` | Path of the Dockerfile |
| `build-env` | empty | Lines `NAME=value` appended to `.env` before the build, for the frontends. Pass `vars`, never secrets: anything in an image is public |
| `scan` | `report` | `report` shows the problems, `block` fails on a secret, a critical flaw that has a fix, or an environment or key file in the image |
| `allowed-files` | empty | Image paths the file check accepts, one per line |
| `health-url` | empty | Health URL; empty means no check and no automatic rollback |
| `health-name` | empty | `serviceName` expected in the health answer (required with `health-url`) |

Example for a frontend that reads its settings at build time:

```yaml
    with:
      image: variamosple
      service: variamospleaws
      build-env: |
        REACT_APP_HOST=${{ vars.REACT_APP_HOST }}
        REACT_APP_URLVMSPROJECTS=${{ vars.REACT_APP_URLVMSPROJECTS }}
```

### Secrets and variables used

Read from the calling repository, its `production` environment or the
organization:

- `vars.ACR_DOKCERHUB`, `secrets.ACRUSERNAME_DOKCERHUB`,
  `secrets.ACRUSERPASSWORD_DOKCERHUB`: Docker Hub;
- `secrets.AWS_HOST`, `secrets.AWS_USERNAME`, `secrets.VARIAMOS_AWS_PEM`: SSH
  access to the server.

At the server restructuring, the SSH access becomes one deploy key per
repository, limited by the server to `deploy <service>`. Only this workflow
changes; the calling repositories keep their file.

## Versions

Callers use the `v1` tag. A change is first tried from one repository with
`@<branch>`, then the `v1` tag is moved. A change that breaks the inputs gets a
new major tag (`v2`).
