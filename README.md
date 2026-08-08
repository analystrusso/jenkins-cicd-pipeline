# Jenkins CI/CD Pipeline — Java Maven App

A Jenkins declarative pipeline that builds a small Spring Boot (Java) app,
packages it as a Docker image, and deploys it to an AWS EC2 instance —
automatically bumping and committing the app's version number on every run.

## What it does, in sequence

Each build runs through five stages, in this order:

1. **Increment version**
   Runs the `build-helper-maven-plugin` and `versions-maven-plugin` against
   `pom.xml` to bump the incremental (patch) version number. Reads the new
   version back out of `pom.xml` and combines it with the Jenkins build
   number to form an image tag, e.g. `1.1.31-42`.

2. **Build jar**
   Runs `mvn clean package` to compile the app and produce the Spring Boot
   executable jar under `target/`.

3. **Build image**
   Builds a Docker image from the repo's `Dockerfile` (which just copies the
   jar into an `amazoncorretto:17-alpine-jdk` base image), logs in to Docker
   Hub, and pushes the image as
   `analystrusso/twn-bootcamp-repo:<version>-<build number>`.

4. **Deploy**
   SSHes into the target EC2 instance, force-removes any existing container
   named `twn-bootcamp`, starts a new container from the image just pushed
   (mapping port `8080`), and checks its status with `docker ps`.

5. **Commit version update**
   Commits the version bump from stage 1 and pushes it back to this repo on
   the current branch, so the version in `pom.xml` stays in sync with what
   was actually built and deployed.

A shared library (`jenkins-shared-library`) is loaded at the top of the
Jenkinsfile but isn't currently invoked by any stage.

## How to run it yourself

### Prerequisites

- A Jenkins controller (any recent version) with:
  - A **Maven** tool installation named exactly `Maven` — configured under
    *Manage Jenkins → Tools*, since the Jenkinsfile references it as
    `tools { maven 'Maven' }`
  - **Docker** installed and usable by the Jenkins agent user
  - The **SSH Agent** plugin
  - Access to the `jenkins-shared-library` repo (or comment out that
    `library` line if you don't need it)
- A Docker Hub account/repo to push images to
- An AWS EC2 instance (or any SSH-reachable Linux host with Docker installed)
  to deploy to
- Four credentials configured in Jenkins with these exact IDs, since the
  Jenkinsfile references them directly:

  | Credential ID       | Type                  | Used for                          |
  |----------------------|-----------------------|------------------------------------|
  | `docker-hub-repo`    | Username/password     | Docker Hub login                   |
  | `github-token`       | Username/password (PAT) | Pushing the version-bump commit  |
  | `github-creds`       | Username/password or SSH | Pulling the shared library     |
  | `ec2-server-key`     | SSH private key       | Deploying to the EC2 host          |

### Setup

1. **Fork or clone this repo.**

2. **Point the pipeline at your own infrastructure.** In the Jenkinsfile,
   update:
   - The Docker Hub image name (`analystrusso/twn-bootcamp-repo`) to your own
     Docker Hub repo
   - The deploy target (`ec2-user@3.80.120.211`) to your own host's user and
     IP/hostname
   - The container name (`twn-bootcamp`) if you want something else

3. **Create a Jenkins Pipeline job** pointing at your repo, with the
   Jenkinsfile as the pipeline definition (*Pipeline script from SCM*).

4. **Add the four credentials** listed above under
   *Manage Jenkins → Credentials*, using the exact IDs shown.

5. **Make sure your EC2 host (or equivalent) has Docker installed** and that
   its security group/firewall allows inbound SSH from your Jenkins agent and
   inbound traffic on port `8080` for the app itself.

6. **Run the pipeline.** On success, the app will be reachable at
   `http://<your-deploy-host>:8080`, and you'll see a new commit in your repo
   bumping the version in `pom.xml`.

### Running the app locally, without Jenkins

If you just want to run the app itself (skip the pipeline entirely):

```bash
mvn clean package
docker build -t twn-bootcamp:local .
docker run -p 8080:8080 twn-bootcamp:local
```

Then visit `http://localhost:8080`.
