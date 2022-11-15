# Hello Chris Tekton
A generic hello world NodeJS application for use in DevOps projects. The Dockerfile uses a NodeJS CentOS image and the CI/CD files are specific to Tekton tool 

This project also shows an example of using the `EventListener` CEL intercepter to trigger builds on specific project folders inside a monorepository.

## Setup

###
Apply the resources found in this repository. The two demo projects are found in /hello-chris and /hello-dave

### Modify Buildah Task
To make this work I used the generic git-clone `ClusterTask` but modified the generic buildah `ClusterTask` to include a seperate workspace called `output` for consumption of the output of the git-clone task.

```
 steps:
  - image: $(params.BUILDER_IMAGE)
    name: build
    resources: {}
    script: |
      buildah --storage-driver=$(params.STORAGE_DRIVER) bud \
        $(params.BUILD_EXTRA_ARGS) --format=$(params.FORMAT) \
        --tls-verify=$(params.TLSVERIFY) --no-cache \
        -f $(params.DOCKERFILE) -t $(params.IMAGE) $(params.CONTEXT)
    volumeMounts:
    - mountPath: /var/lib/containers
      name: varlibcontainers
    workingDir: $(workspaces.output.path)  <------ changed this to output from source workspace
  - image: $(params.BUILDER_IMAGE)
    name: push
    resources: {}
    script: |
      buildah --storage-driver=$(params.STORAGE_DRIVER) push \
        $(params.PUSH_EXTRA_ARGS) --tls-verify=$(params.TLSVERIFY) \
        --digestfile $(workspaces.source.path)/image-digest $(params.IMAGE) \
        docker://$(params.IMAGE)
    volumeMounts:
    - mountPath: /var/lib/containers
      name: varlibcontainers
    workingDir: $(workspaces.source.path)
  - image: $(params.BUILDER_IMAGE)
    name: digest-to-results
    resources: {}
    script: cat $(workspaces.source.path)/image-digest | tee /tekton/results/IMAGE_DIGEST
  volumes:
  - emptyDir: {}
    name: varlibcontainers
  workspaces:
  - name: source
  - name: output  <------- added output workspace
```

NOTE: When the operator updates it will override these changes, I recommend changing this to a `Task` managed within your own namespace

### Integrate with GitLab
A secret needs to be generated for to access a private repository on GitLab
`oc create secret generic git-ssh --from-file=ssh-privatekey --from-file=known_hosts --type=kubernetes.io/ssh-auth`

Create a webhook on your GitLab project that uses the link of a `Route` linked to the `Service` that was generated in your namespace after applying the `EventListener` resource. You will need to create the `Route` on your own as Tekton does not generate this for you at the time of this writing.

This example uses GitLab, but can be easily modified for GitHub by changing the CEL expression field in the eventlistener.yaml

## Running
For the `EventListener` to work correctly the user commiting their code must add the name of the project as the first part of their commit, for example,
`git commit -am 'hello-chris updating README'

To trigger hello-dave it would be
`git commit -am 'hello-dave updating README'

The above assumes that the code and `Dockerfile` exist on the directory layer of hello-chris and hello-dave, this can be modifed by changing the Dockerfile path in the `Pipeline` and by modifying the `Dockerfile` ADD commands.

This example also assumes that hello-dave can use the same `Pipeline` as hello-chris, this can be modified by changing the template in the `EventListener` and adding a new `TriggerTemplate` that has a new `PipelineRun` definition for hello-dave if the build process was different between the two projects. For example, if one was a backend application using java and the other was a front end project using nodejs.
Feel free to pull, fork, and modify as desired.
