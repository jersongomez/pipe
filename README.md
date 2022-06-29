apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  annotations:
    tekton.dev/categories: Build Tools
    tekton.dev/pipelines.minVersion: 0.12.1
    tekton.dev/platforms: 'linux/amd64,linux/s390x,linux/ppc64le'
    tekton.dev/tags: build-tool
  name: nodejs
  labels:
    app.kubernetes.io/version: '0.2'
    operator.tekton.dev/provider-type: community
spec:
  description: This Task can be used to run a Nodejs command.
  params:
    - default: >-
        registry.redhat.io/rhel8/nodejs-14-minimal:1
      description: Nodejs base image
      name: NODEJS_IMAGE
      type: string
    - default:
        - run
      description: nodejs goals to run
      name: GOALS
      type: array
    - default: .
      description: >-
        The context directory within the repository for sources on which we want
        to execute maven goals.
      name: CONTEXT_DIR
      type: string
    - default: /usr/bin/npm
      name: COMMAND
      type: string
  steps:
    - args:
        - $(params.GOALS)
      command:
        - $(params.COMMAND)
      image: $(params.NODEJS_IMAGE)
      name: nodejs-goals
      resources: {}
      workingDir: $(workspaces.source.path)/$(params.CONTEXT_DIR)
  workspaces:
    - description: The workspace consisting of maven project.
      name: source
