Development notes

To run the input service and the local mock analysis service together for development:

1. Start the analysis service (this folder):

   mvn -f analysis/pom.xml spring-boot:run

2. Start the main input service (root project) as usual:

   mvn spring-boot:run

By default the input service Feign client property `analysis.service.url` is read from `application.properties` or the `ANALYSIS_SERVICE_URL` env var. For local testing it defaults to `http://diet-analysis-service:8082`. To point to the local analysis service set:

export ANALYSIS_SERVICE_URL=http://localhost:8082

or on Windows PowerShell:

$env:ANALYSIS_SERVICE_URL = 'http://localhost:8082'

Then run the input service.

AI photo nutrition analysis

The analysis service now calls the OpenAI Responses API for food-photo nutrition estimates.
Set an API key before starting the analysis service:

Windows PowerShell:

    $env:OPENAI_API_KEY = 'your-api-key'
    $env:OPENAI_MODEL = 'gpt-4.1-mini'
    mvn -f analysis/pom.xml spring-boot:run

If `OPENAI_API_KEY` is empty, or if the AI API call fails, the service returns the existing
mock result by default so demos and CI can still run. To fail fast instead, set:

    $env:OPENAI_MOCK_FALLBACK = 'false'

VS Code local API key setup

For VS Code, you can use a local-only properties file instead of typing the key in
PowerShell every time.

1. Open `analysis/src/main/resources/application-local.properties`.
2. Replace `sk-proj-your-api-key-here` with your real OpenAI API key.
3. In VS Code, open Run and Debug.
4. Make sure the opened folder is the project root `diet_input-main`, not the
   `analysis` folder.
5. Wait until VS Code Java extensions finish importing the Maven projects.
6. Start `Run Diet AI Services`.
7. Open `http://localhost:8081`.

If VS Code says `ClassNotFoundException` for one of the application classes,
open `diet-ai.code-workspace` instead, then run `Java: Clean Java Language Server
Workspace` from the command palette and reload the window.

`application-local.properties` is ignored by git, so the real key should not be
committed. The analysis Docker build also excludes this local file so your key
does not get copied into a container image. Keep `application-local.properties.example`
as the shareable template.

Gateway service

The `gateway` folder contains a Spring Cloud Gateway service. It listens on port 8080.

Local routes:

- `/api/analysis/**` -> analysis service
- `/api/**` -> input service
- `/**` -> input service UI and static files

Run locally:

    mvn -f gateway/pom.xml spring-boot:run

For local execution, start services in this order:

    mvn -f analysis/pom.xml spring-boot:run
    mvn spring-boot:run
    mvn -f gateway/pom.xml spring-boot:run

Then open:

    http://localhost:8081

Continuous integration and deployment

This repository uses GitHub Actions CI in `.github/workflows/ci.yml` for build and test.
The `ci.yml` workflow runs on push to `main` and `feature/**`, and on pull requests targeting `main`.

Deployment is handled by Argo CD, so CI builds and pushes Docker images for the services while Argo CD performs the deployment.

The CI workflow builds and verifies:

- input service
- analysis service
- gateway service
- eureka server

Images are pushed to Docker Hub and can be deployed by Argo CD from the image registry.

Istio service mesh

Kubernetes deployment includes Istio manifests in `base/istio.yaml`.

Istio is used for cluster traffic control:

- External traffic enters through `diet-public-gateway`.
- Istio routes public traffic to `diet-gateway-service`.
- `input-service` calls `diet-analysis-service` through the mesh.
- `analysis-service-circuit-breaker` limits pending requests and ejects unhealthy
  analysis service endpoints when repeated 5xx failures occur.

The `book-dev` namespace enables automatic sidecar injection:

    labels:
      istio-injection: enabled

Local VS Code execution does not use Istio. It runs the same service chain directly:

    browser -> gateway(local 8081) -> input(local 8080) -> analysis(local 8082)

Eureka service discovery

The shared `diet_analysis-main` zip includes Istio manifests, but it does not
include Eureka dependencies or Eureka client/server configuration. This project
therefore adds a dedicated Eureka server in the `eureka` folder.

Local VS Code execution now starts services in this order:

    eureka(8761) -> analysis(8082) -> input(8080) -> gateway(8081)

In local mode, `analysis` registers itself as `diet-analysis-service` in Eureka.
The input service uses Feign with the service name `diet-analysis-service`, so it
can call the analysis service through Eureka discovery.

In Kubernetes, the services still run as Kubernetes Services and Istio controls
traffic policies between `input-service` and `diet-analysis-service`.
test clid $ (Get-Date)
