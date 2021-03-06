apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  annotations:
    tekton.dev/displayName: s2i nginx
    tekton.dev/pipelines.minVersion: '0.19'
    tekton.dev/tags: 's2i, nginx, workspace'
  name: s2i-nginx
  labels:
    app.kubernetes.io/version: '0.1'
    operator.tekton.dev/provider-type: redhat
spec:
  description: >-
    s2i-nginx task clones a Git repository and builds and pushes a container
    image using S2I and a nginx builder image.
  params:
    - default: 1.18-ubi8
      description: The tag of nginx imagestream for nginx version
      name: VERSION
      type: string
    - default: .
      description: The location of the path to run s2i from.
      name: PATH_CONTEXT
      type: string
    - default: 'true'
      description: >-
        Verify the TLS on the registry endpoint (for push/pull to a non-TLS
        registry)
      name: TLSVERIFY
      type: string
    - description: Location of the repo where image has to be pushed
      name: IMAGE
      type: string
    - default: ''
      description: Nginx config file path inside source
      name: CONFIG_FILE_PATH
      type: string
    - default: >-
        registry.redhat.io/rhel8/buildah@sha256:6a68ece207bc5fd8db2dd5cc2d0b53136236fb5178eb5b71eebe5d07a3c33d13
      description: The location of the buildah builder image.
      name: BUILDER_IMAGE
      type: string
  results:
    - description: Digest of the image just built.
      name: IMAGE_DIGEST
  steps:
    - args:
        - |-
          echo "CONFIG FILE PATH = $(params.CONFIG_FILE_PATH)"

          [[ '$(params.CONFIG_FILE_PATH)' != "" ]] &&
            cp '$(params.CONFIG_FILE_PATH)' '$(params.PATH_CONTEXT)/'

          echo "------------------------------"
          ls -la '$(params.PATH_CONTEXT)/'
          echo "------------------------------"
      command:
        - /bin/sh
        - '-c'
      image: >-
        registry.redhat.io/ocp-tools-4-tech-preview/source-to-image-rhel8@sha256:ba51e5e74ff5a29fd429b0bb77bc2130e5284826a60d042fc4e4374381a7a308
      name: config-file
      resources: {}
      volumeMounts:
        - mountPath: /gen-source
          name: gen-source
      workingDir: /gen-source
    - command:
        - s2i
        - build
        - $(params.PATH_CONTEXT)
        - >-
          image-registry.openshift-image-registry.svc:5000/openshift/nginx:$(params.VERSION)
        - '--as-dockerfile'
        - /gen-source/Dockerfile.gen
      image: >-
        registry.redhat.io/ocp-tools-4-tech-preview/source-to-image-rhel8@sha256:ba51e5e74ff5a29fd429b0bb77bc2130e5284826a60d042fc4e4374381a7a308
      name: generate
      resources: {}
      volumeMounts:
        - mountPath: /gen-source
          name: gen-source
      workingDir: $(workspaces.source.path)
    - command:
        - buildah
        - bud
        - '--storage-driver=vfs'
        - '--tls-verify=$(params.TLSVERIFY)'
        - '--layers'
        - '-f'
        - /gen-source/Dockerfile.gen
        - '-t'
        - $(params.IMAGE)
        - .
      image: $(params.BUILDER_IMAGE)
      name: build
      resources: {}
      volumeMounts:
        - mountPath: /var/lib/containers
          name: varlibcontainers
        - mountPath: /gen-source
          name: gen-source
      workingDir: /gen-source
    - command:
        - buildah
        - push
        - '--storage-driver=vfs'
        - '--tls-verify=$(params.TLSVERIFY)'
        - '--digestfile=$(workspaces.source.path)/image-digest'
        - $(params.IMAGE)
        - 'docker://$(params.IMAGE)'
      image: $(params.BUILDER_IMAGE)
      name: push
      resources: {}
      volumeMounts:
        - mountPath: /var/lib/containers
          name: varlibcontainers
      workingDir: $(workspaces.source.path)
    - image: $(params.BUILDER_IMAGE)
      name: digest-to-results
      resources: {}
      script: >-
        cat $(workspaces.source.path)/image-digest | tee
        /tekton/results/IMAGE_DIGEST
  volumes:
    - emptyDir: {}
      name: varlibcontainers
    - emptyDir: {}
      name: gen-source
  workspaces:
    - mountPath: /workspace/source
      name: source
