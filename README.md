# VProfile Application

VProfile is a Java web application for account registration, sign-in, profile viewing and editing, and user administration. It is implemented with Spring MVC and JSP, packaged as a WAR, and deployed to Tomcat. The application also integrates with MySQL, Memcached, RabbitMQ, and Elasticsearch.

This repository contains the application source, container build, and GitHub Actions delivery workflow. Helm deployment configuration is maintained separately in the `vprofile-helm` repository.

## Service Overview

### Technology

- Java 21 is used by CI and the Docker build image. The Maven compiler source and target properties in `pom.xml` are currently set to Java 17.
- Maven builds a WAR artifact named `vprofile-v2.war`.
- Spring Framework 6 / Spring MVC with JSP views, Spring Security, Spring Data JPA, and Hibernate.
- Tomcat 10 runs the WAR as the root web application on port 8080.
- MySQL stores account data; Memcached is used by user lookup; RabbitMQ and Elasticsearch are integrated by application controllers and services.

### Main User Flows

- Register an account and authenticate through the Spring Security login form.
- View and update account profiles, and browse users.
- Retrieve user data by ID, using Memcached for cache lookup.
- Use the RabbitMQ status/interaction route and Elasticsearch user indexing and document operations.
- Upload files through the upload controller.

Representative routes in the current controllers include `/registration`, `/login`, `/welcome`, `/users`, `/users/{id}`, `/user/{username}`, `/user/rabbit`, `/user/elasticsearch`, and `/upload`. Consult the controller classes for exact methods, authorization behavior, and parameters.

### Runtime Dependencies and Configuration

The default values in `src/main/resources/application.properties` point to these service hosts:

| Dependency | Default host | Port |
| --- | --- | ---: |
| MySQL | `vprodb` | 3306 |
| Memcached active | `vprocache01` | 11211 |
| RabbitMQ | `vpromq01` | 5672 |
| Elasticsearch | `localhost` | 9300 |

The Spring XML configuration under `src/main/webapp/WEB-INF` wires the data source/JPA repositories, Spring Security, and RabbitMQ. Adjust service endpoints and credentials for the target environment before running the application.

**Security note:** Spring Security uses BCrypt for password encoding, but CSRF protection is disabled in the current configuration. The checked-in application properties also include database, RabbitMQ, and sample administrator credentials. Review the security configuration, replace credentials with externally injected configuration, and rotate any credentials that have been used outside a local environment. Do not treat the checked-in values as production secrets.

## Build and Run

Requirements: JDK 21 and Maven 3.9 or later. From the repository root:

```sh
mvn clean verify
```

The WAR is created at `target/vprofile-v2.war`. Build and run the Tomcat image with Docker:

```sh
docker build -f Docker-files/app/multistage/Dockerfile -t vprofileapp:local .
docker run --rm -p 8080:8080 vprofileapp:local
```

The container expects the configured backing services to be reachable. Running only the application container without them will not provide a fully functional deployment. Open `http://localhost:8080/` after starting the app and its dependencies.

`Docker-files/db/Dockerfile` describes a MySQL image that initializes from the included SQL backup. `Docker-files/web` contains an Nginx reverse-proxy configuration for a service named `vproapp`. These files are supporting container assets; the CI workflow currently builds only the application image.

## CI and GitOps Delivery

The workflow is [.github/workflows/ci.yml](.github/workflows/ci.yml). Its branch filters are limited to `main`:

| Event | Jobs |
| --- | --- |
| Push to a feature or other non-main branch | No workflow run |
| Pull request targeting `main` | `build-and-sonar` |
| Push to `main` (including a merge) | `docker-build-push`, followed by `update-helm` |

### Pull Request Checks

The PR job checks out full Git history, configures JDK 21, caches Maven dependencies and the Sonar scanner cache, then runs Maven verification and Checkstyle report generation. It scans with the self-hosted SonarQube server using the existing `sonar-project.properties` and checks the quality gate, with a 10-minute polling timeout. A failed job blocks merging only when `build-and-sonar` is configured as a required status check in the repository's branch protection or ruleset.

### Main-Branch Image and Helm Update

After a push to `main`, the Docker job configures AWS credentials, creates the ECR repository if it is missing, and builds with `Docker-files/app/multistage/Dockerfile`. It pushes two tags to ECR:

- The first seven characters of the Git commit SHA.
- `latest`.

The job passes the ECR registry, full image name, and image tag to `update-helm` using job outputs. That job checks out the separate Helm repository, updates `app.image` and `app.tag` in `helm/vprofile/values.yaml` using `yq`, then commits and pushes the change to the Helm repository's `main` branch.

The Helm clone URL is constructed from the current repository owner and `HELM_REPO_NAME`. The Helm repository therefore needs to be accessible under the same GitHub owner, and must contain `helm/vprofile/values.yaml` with an `app` mapping. The resulting values are equivalent to:

```yaml
app:
  image: <AWS-account>.dkr.ecr.<region>.amazonaws.com/<repository>
  tag: <seven-character-commit-sha>
```

The chart/deployment system watching the Helm repository is responsible for applying that change to a cluster; this workflow updates Git desired state and does not itself run `helm upgrade` or connect to Kubernetes.

## GitHub Configuration

Configure these repository or organization variables:

| Variable | Used for |
| --- | --- |
| `AWS_REGION` | ECR region, expected to be `us-east-1` for the configured repository |
| `ECR_REPOSITORY` | ECR image repository name, expected to be `vprofileappimg` |
| `HELM_REPO_NAME` | Separate Helm repository name, expected to be `vprofile-helm` |
| `SONAR_HOST_URL` | Base URL of the self-hosted SonarQube server |

Configure these GitHub Actions secrets:

| Secret | Used for |
| --- | --- |
| `SONAR_TOKEN` | SonarQube scan and quality-gate authentication |
| `AWS_ACCESS_KEY_ID` | AWS authentication for ECR operations |
| `AWS_SECRET_ACCESS_KEY` | AWS authentication for ECR operations |
| `HELM_REPO_USER` | GitHub username in the Helm clone URL |
| `GITOPS_PAT` | Personal access token allowing clone and push to the Helm repository |

Grant the AWS identity permission to describe/create the configured ECR repository and authenticate, upload layers, and publish image tags. The GitHub token needs read and write access to the Helm repository. Prefer short-lived AWS credentials through GitHub OIDC over long-lived access keys when the environment is ready to support OIDC.

## Repository Map

- `src/main/java/com/visualpathit/account`: Spring MVC controllers, models, repositories, services, security, and utilities.
- `src/main/resources`: application configuration and SQL resources.
- `src/main/webapp/WEB-INF/views`: JSP pages.
- `Docker-files/app/multistage/Dockerfile`: Maven WAR build and Tomcat runtime image used by CI.
- `Docker-files/db`: MySQL container assets and database backup.
- `Docker-files/web`: Nginx reverse-proxy configuration.
- `sonar-project.properties`: SonarQube project identity, source, binary, and report paths.
- `.github/workflows/ci.yml`: PR quality checks and main-branch GitOps image publication flow.
