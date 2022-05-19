![jenkins-a-love-hate-relatiosnhip](img/title.png)

Purpose of this repo is to demonstrate various Jenkins setups and to provide step-by-step guides on how to configure them locally.

# Jenkins Setup(s)
* [Jenkins with WSL](doc/wsl.md)
* TODO: [Jenkins with Docker](doc/docker.md)
* TODO: [Jenkins with Docker-Compose](doc/docker-compose.md)
* TODO: [Jenkins with Kubernetes](doc/kubernetes.md)

# Let's Build Something

Now, that we have a Jenkins, let's build something. We are going to have a couple of requirements before we can actually build, but if you follow the guide it should be easy.

## Prerequisites
* Follow [wsl-from-internet](./doc/wsl/wsl-from-internet.md) guide on how to make WSL accessible on the internet.
* Have a [GitHub](https://github.com/) account
* Follow [GitHub Personal Access Token](./doc/credentials/github-personal-access-token.md) to create one. 
* Fork the [simple-java-maven-app](https://github.com/jenkins-docs/simple-java-maven-app)
* Follow [Build Tools Guide](./doc/build-tools.md) to configure the necessary build tools for Jenkins.

## 

* [freestyle](./doc/jobs/freestyle.md)
* TODO [pipeline](./doc/jobs/pipeline.md)
  * TODO [with raspberry pi](./doc/jobs/raspberry-pi.md)
  * TODO [with amd64](./doc/jobs/amd64.md)
* TODO [jenkinsfile](./doc/jobs/jenkinsfile.md)
* TODO [jenkins library](./doc/jobs/jenkins-library.md)