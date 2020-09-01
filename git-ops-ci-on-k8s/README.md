# git-ops ci on k8s

The CI system's state is per git commit and represented as annotations.

``` bash
git tag --points-at c392e7b
ci/c392e7b/docker-build
ci/c392e7b/test-result-ok
ci/c392e7b/deployed
```

Since every tag-ref can only be used once, a unique identifier is part of its name. All tags are annotated and the message may include json data.

``` bash
git tag -a ci/$(git rev-parse --short HEAD)/docker-build -m '{ "image": "my-docker-registry.com/my-image:latest" }'
```

Relation between kubernetes-jobs and git-tags are defined in a TriggeredJob resource

``` yaml
apiVersion: ci/v1beta1
kind: TriggeredJob
metadata:
  namespace: my-pipeline
  name: create-docker-image
spec:
  trigger:
    all:
    - git:
        repoUrl: https://my-awesome/repository.git
        hasAllTags:
        - /release

  # jobTemplate equals batch/v1beta1/CronJob.spec.jobTemplate
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: build-image
            image: my-image-builder
          restartPolicy: OnFailure
```

Build steps are defined in docker-image, e.g. with multi-stage builds. After a step, a new tag is pushed, eventually triggering the next step.

``` Dockerfile
# Clone
FROM git AS src
RUN git clone $REPO_URL
RUN git checkout ci/*******/release

# Resolve dependencies
FROM node AS build
COPY --from=src ...
RUN npm install

# Push results
FROM docker
COPY --from=build ...
RUN docker push ...

git tag -a ci/*******/docker-build -a '{ "image":  "my-new-image" }'
git push
```
