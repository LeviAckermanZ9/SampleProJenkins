# SampleProJenkins

A small static website packaged as an Nginx container, with a Jenkins pipeline for Docker Hub publishing and optional deployment to an EC2 instance.

## How it fits together

```text
GitHub push → Jenkins → build container → Docker Hub → (optional) EC2
```

Each successful pipeline build publishes an immutable image tagged with the Git commit, plus a moving `latest` tag. EC2 deployment is opt-in per build.

## Run locally with Docker

```sh
docker build -t sampleprojenkins .
docker run --rm -p 8080:80 sampleprojenkins
```

Open `http://localhost:8080`.

## Prepare Docker Hub

1. Create a Docker Hub repository named `sampleprojenkins` (a public repository is simplest for the EC2 pull step).
2. Create a Docker Hub access token for Jenkins.
3. In Jenkins, add a **Username with password** credential with ID `dockerhub-creds`. Use the Docker Hub username and access token.

## Prepare Jenkins

The Jenkins build agent needs Git, Docker CLI, access to a Docker daemon, and a Linux shell. Install the **Pipeline** and **Credentials Binding** plugins. Give the Jenkins agent permission to run Docker commands.

Create a Pipeline job pointing to this GitHub repository and set its script path to `Jenkinsfile` (or create a Multibranch Pipeline). On the first run, replace the sample `DOCKERHUB_REPO` value with your actual Docker Hub `username/sampleprojenkins` repository.

The pipeline publishes the commit-tagged image and `latest` to Docker Hub. Credentials stay in Jenkins; never put tokens or private keys in this repository.

## Optional: deploy to AWS EC2

1. Launch a Linux EC2 instance and install Docker on it. Create a non-root deploy user that can run Docker.
2. Configure the instance security group to allow inbound HTTP on port 80. Restrict SSH (port 22) to your trusted IP addresses.
3. Add the EC2 private key to Jenkins as an **SSH Username with private key** credential with ID `ec2-ssh-key`. Set its username to the EC2 deploy user.
4. When starting a build, enter the instance's public DNS name or IP address in the `EC2_HOST` parameter and enable `DEPLOY_TO_EC2`.
5. Add the EC2 server's verified SSH host key to the Jenkins agent user's `~/.ssh/known_hosts`. The pipeline intentionally requires strict host key checking.
6. Run a build with **Deploy to EC2** enabled.

The image repository must be public so EC2 can pull without credentials. For a private Docker Hub repository, configure Docker Hub read-only credentials securely on the instance before deployment. The site will be available at `http://<EC2 public address>/` after the build completes.

## Pipeline credentials and settings

| Jenkins item | Type | Purpose |
| --- | --- | --- |
| `dockerhub-creds` | Username with password | Docker Hub username and access token |
| `ec2-ssh-key` | SSH Username with private key | EC2 deploy user's SSH identity |
| `EC2_HOST` | Build parameter | EC2 public DNS name or IP; only needed for deployment |

The EC2 stage is skipped by default. The Jenkins agent must be able to reach Docker Hub, the GitHub repository, and (when deploying) SSH on the EC2 instance.
