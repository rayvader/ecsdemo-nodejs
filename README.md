![Build Status](https://codebuild.us-east-2.amazonaws.com/badges?uuid=eyJlbmNyeXB0ZWREYXRhIjoidDRkc2xFKzlVaVpRZXRiK1VrV3crbVNQMnJBNHdzQjQrVlQwaXdqVHdlYW94RU9jODN0R0I4WGJieEJLdjRldktxN3pCMWNTbW9YVUVPUmNzcVFCd0FjPSIsIml2UGFyYW1ldGVyU3BlYyI6Ikg0S29yNDFQUEVHMWVoN2wiLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D&branch=master)

# Amazon ECS Workshop

This is part of an Amazon ECS workshop at https://ecsworkshop.com

`ecsdemo-nodejs` is the Node.js backend service of the workshop. It is a small Express app that reports which container IP and Availability Zone served the request, and which commit it was built from.

## Features

- `GET /` and `GET /nodejs`: plain text, `Node.js backend: Hello! from <IP> in AZ-<zone> commit <hash>`
- `GET /nodejs/api`: the same data as JSON (`from`, `message`, `commit`)
- `GET /health`: health check for load balancers (`express-healthcheck`)
- Apache-style access logs (`morgan`)
- On start-up, `startup.sh` works out the orchestrator (ECS or Kubernetes) and the AZ from the container IP, EC2 instance metadata, or, on Fargate, the task metadata endpoint plus `ec2:DescribeSubnets`

## Tech stack

- Node.js, Express 4
- Mocha + Supertest for tests
- Docker (Alpine 3.6 base image)
- Deploy options: AWS CDK (Python, CDK v1), ECS CLI (`docker-compose.yml` + `ecs-params.yml`), Mu (`mu.yml`), Kubernetes manifests, AWS CodeBuild build specs

## Project structure

```
.
├── server.js                 # Express app (port 3000)
├── test/test.js              # Mocha/Supertest tests
├── package.json
├── code_hash.txt             # Commit hash shown in responses (written by CI)
├── startup.sh                # Container entrypoint: detects IP/AZ, then runs node
├── startup-cdk.sh            # Same, also exports AWS_DEFAULT_REGION from REGION
├── Dockerfile                # Main image
├── Dockerfile.cdk            # Image for the CDK deployment (uses startup-cdk.sh)
├── cdk/                      # CDK v1 app: Fargate service with Cloud Map discovery
├── docker-compose.yml        # ECS CLI compose file
├── ecs-params.yml            # ECS CLI params (environment specific)
├── ecs-params.yml.template   # Template for ecs-params.yml
├── mu.yml                    # Mu service definition
├── kubernetes/               # Deployment and Service manifests
├── buildspec.yml             # CodeBuild: writes code_hash.txt and imagedefinitions.json
├── buildspec-test.yml        # CodeBuild: smoke test against BASE_URL
└── buildspec-dockerhub.yml   # CodeBuild: build and push images to Docker Hub
```

## Prerequisites

- Node.js and npm (local runs)
- Docker (container builds)
- For deployment: the ECS workshop platform (VPC, ECS cluster, Cloud Map namespace) and the tool for your chosen path: AWS CDK v1 + Python 3, ECS CLI, Mu or `kubectl`

## Install and run locally

```bash
npm install
node server.js        # listens on http://localhost:3000
```

`server.js` reads `code_hash.txt` at start-up, so keep that file in place. Run outside a container, `IP` and `AZ` are unset and show as `undefined` in responses.

## Test

```bash
npm install           # includes dev dependencies
npm test              # mocha: /, /health and a 404 case
```

## Docker

```bash
docker build -t ecsdemo-nodejs .
docker run --rm -p 3000:3000 ecsdemo-nodejs
```

Build with `--build-arg NODE=development` to set `NODE_ENV=development`. The image includes the AWS CLI and `netaddr`, which `startup.sh` uses on Fargate. A `HEALTHCHECK` calls `/health/` every 10 seconds.

## Configuration

Environment variable names used by the app and scripts:

| Name | Used by | Meaning |
|---|---|---|
| `IP`, `AZ` | `server.js` | Set by `startup.sh`; shown in responses |
| `ECS_CONTAINER_METADATA_URI` | `startup.sh` | Set by ECS; used to find the task IP on Fargate |
| `REGION` | `startup-cdk.sh` | Set by the CDK task definition; exported as `AWS_DEFAULT_REGION` |
| `NODE_ENV` | Docker | From build arg `NODE` (default `production`) |
| `AWS_ACCOUNT_ID`, `AWS_DEFAULT_REGION` | `cdk/app.py` | Target account and region for the CDK stack |
| `AWS_REGION` | `docker-compose.yml` | CloudWatch Logs region |

## Deploy

Follow the matching chapter of the [ECS workshop](https://ecsworkshop.com). In short:

### AWS CDK (Fargate)

```bash
cd cdk
pip install -r requirements.txt
export AWS_ACCOUNT_ID=<account-id> AWS_DEFAULT_REGION=<region>
cdk synth
cdk deploy
```

The stack `ecsworkshop-nodejs` imports the `ecsworkshop-base/BaseVPC` VPC and the `NSNAME`, `NSARN`, `NSID` and `ECSClusterName` exports from the workshop platform stack. It runs one Fargate task (256 CPU, 512 MiB) from `brentley/ecsdemo-nodejs:cdk`, registered in Cloud Map as `ecsdemo-nodejs`.

### ECS CLI

Copy `ecs-params.yml.template` to `ecs-params.yml` and fill in the task execution role, three subnets and a security group for your environment, then deploy `docker-compose.yml` with `ecs-cli compose service up`.

### Kubernetes

```bash
kubectl apply -f kubernetes/deployment.yaml -f kubernetes/service.yaml
```

The Deployment runs `brentley/ecsdemo-nodejs:latest` on port 3000; the Service exposes it on port 80.

### CI (CodeBuild)

- `buildspec.yml` writes `code_hash.txt` and `imagedefinitions.json` (container `example-backend`) from the commit hash. It needs `REPOSITORY_URI`.
- `buildspec-dockerhub.yml` builds the main and CDK images and pushes them to Docker Hub. It reads Docker Hub credentials from SSM Parameter Store (`docker_hub_username`, `docker_hub_password`).
- `buildspec-test.yml` reads `BASE_URL` from `mu-env.sh` and checks it responds.

## License

See [LICENSE](LICENSE).
