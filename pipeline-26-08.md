# Docker
# Build a Docker image
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- main

resources:
- repo: self

variables:
  tag: '$(Build.BuildId)'
  imageName: nodeapplication

stages:
- stage: AppBuild
  displayName: "Build and Package App"
  jobs:
  - job: Build_the_app
    pool:
      name: azure
    steps:
    - script: |
        npm ci
      displayName: "Install dependencies"
    - script: |
        npm pack
        mv *.tgz $(Build.ArtifactStagingDirectory)/app.tgz
      displayName: "Packaging the app"
    - publish: $(Build.ArtifactStagingDirectory)/app.tgz
      artifact: app-package
      displayName: "Publish app.tgz"
- stage: DockerImageBuild
  displayName: Build image
  dependsOn: AppBuild
  jobs:
  - job: Docker_Image
    displayName: "Build Docker Image"
    pool:
      name: azure
    steps:
    - download: current
      artifact: app-package
      displayName: "Download app artifact"
    - task: Docker@2
      displayName: Build and Push Docker image to Docker Hub
      inputs:
        command: buildAndPush 
        containerRegistry: 'dockerhub'
        dockerfile: '$(Build.SourcesDirectory)/Dockerfile'
        buildContext: '$(Pipeline.Workspace)/app-package'
        repository: 'ayat93/nodeapp'
        tags: $(tag)
    
- stage: Deploy
  displayName: "Run Container on Green VM"
  dependsOn: DockerImageBuild
  jobs:
  - job: RunContainer
    pool:
      name: green
    steps:
    - script: |
        echo "Stopping old container if running..."
        docker rm -f nodeapp || true

        echo "Pulling image from Docker Hub..."
        docker pull ayat93/nodeapp:$(tag)


        echo "Starting new container..."
        docker run -d -p 80:3000 --name nodeapp $(imageName):$(tag)
      displayName: "Deploy container on Green VM"
