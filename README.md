# openshift-dockerfile-exercise
Exercise in building and deploying from a simple Dockerfile in Openshift 4.

## Goals
The purpose of this exercise is to learn basics of:
* Build an application from source (a git repo) in Openshift, using the Dockerfile build strategy
* Deploy an application in Openshift
* Perform basic status checks on the build and dployment

## How to build and run
1. Fork this repo into your own git account. Make the fork public (to avoid dealing with Git credentials in Openshift)
1. Clone your fork, e.g: `$ git clone git@github.com:svejk-ciber/openshift-dockerfile-example.git`
1. Log in to Openshift
   `$ oc login -u developer -p developer` # assuming Katacoda.
1. Create a new project for the application:
   `$ oc new-project docker-build`
1. Create a new app:
  ```$ oc new-app --name sleep  https://github.com/svejk-ciber/openshift-dockerfile-example.git
  ...
  --> Creating resources ...
    imagestream.image.openshift.io "bash" created
    imagestream.image.openshift.io "sleep" created
    buildconfig.build.openshift.io "sleep" created
    deploymentconfig.apps.openshift.io "sleep" created
--> Success
    Build scheduled, use 'oc logs -f bc/sleep' to track its progress.
    Run 'oc status' to view your app.
  ```
  
   You can also use the argument `--strategy=docker` to be explicit, but in this case there is no ambguity, 
   since there are no other source files present than the Dockerfile, so Openshift should not choose a different build strategy than _Docker_.
1. `$ oc status`
```
...
dc/sleep deploys istag/sleep:latest <-
  bc/sleep docker builds https://github.com/svejk-ciber/openshift-dockerfile-example.git#solution on istag/bash:5.0.11
  deployment #1 deployed 7 minutes ago - 1 pod


3 infos identified, use 'oc status --suggest' to see details.
```
  Should not indicate any problems, apart from missing Kubernetes probes. Folloe the tip about the `--suggest` 
  parameter to verify this.
  
1. Briefly review the resources create by `new-app`:
`$ oc get all`
What resources did `new-app` create?

1. Check the build log for errors:
  ```$ oc logs bc/sleep
  ...
  Push successful
  ```
  Observe that Openshift runs a Dockerfile build, and adds some metadata to the build image with
  `ENV` and `LABEL`instructions.
1. Wait for the pod to become running:
  ```$ oc get pods -w
  NAME             READY   STATUS      RESTARTS   AGE
sleep-1-8j65k    1/1     Running     0          12m
sleep-1-build    0/1     Completed   0          13m
sleep-1-deploy   0/1     Completed   0          12m
  ```
  The builder and deployer pods are done, and we are left with a single running pod, the one with just a Docker 
  image suffix in the name.
 
1. View the application log:
```
...
Sleep.
Sleep.
Sleep.
```

1. Review resources created by Openshift:
1.1 Look at the BuildConfiguration:
   ```$ oc describe bc sleep
Name:           sleep
Namespace:      docker-build
Created:        16 minutes ago
Labels:         app=sleep
...   
Strategy:       Docker
URL:            https://github.com/svejk-ciber/openshift-dockerfile-example.git
Ref:            solution
From Image:     ImageStreamTag bash:5.0.11
Output to:      ImageStreamTag sleep:latest
...
   ``` 
Note that the build strategy has been set to `Docker`, since the source is an unambigious Dockerfile,
and that the label `app` has been added with the value of the argument we gave to `oc new-app` previously.
Also observe that the build configuration input and outputs looks as expected.

1.2 Inspect the image stream for the build image
```
$ oc describe is sleep
Name:                   sleep
Namespace:              docker-build
Created:                14 seconds ago
Labels:                 app=sleep
Annotations:            openshift.io/generated-by=OpenShiftNewApp
Image Repository:       default-route-openshift-image-registry.apps-crc.testing/docker-build/sleep
Image Lookup:           local=false
Tags:                   <none>
```
Observe that the built image is located in Openshift's internal registry. 

1.3 Inspect the deployment Configuration
```
$ oc describe dc sleep
Name:           sleep
Namespace:      docker-build
Created:        4 minutes ago
Labels:         app=sleep
...
Replicas:       1
Triggers:       Config, Image(sleep@latest, auto=true)
Strategy:       Rolling
Template:
Pod Template:
  Labels:       app=sleep
                deploymentconfig=sleep
  Annotations:  openshift.io/generated-by: OpenShiftNewApp
  Containers:
   sleep:
    Image:              image-registry.openshift-image-registry.svc:5000/docker-build/sleep@sha256:237b14f609d3ab452f7f45ba149119a5d1c97324757e8b29a0acb55b4bc8752f
...
```
Note that the deployment config has a trigger in the image stream we looked at before. If the image and thus 
image stream changes, a new deployment is performed. It is possible to use the hash of the Docker image to 
identify the image version during troubleshooting and rollbacks (since the `latest` tag gets replaced every 
time we push a new image version to the internal registry)

1. Check the log of the running container:
  `$ oc logs sleep-1...`
1. Observe that `new-app` has created the following resources: one build configuration, one deployemnt configuration
one replication controller, one build and two image streams (oc get bc|dc|is|rc|build|pod...)

2. Change the application

2.1 Edit the `CMD` instruction in the Dockerfile to display a counter. The result should look similar to this
```
FROM bash:5.0.11

CMD ["bash", "-c", "while true; do (( i++ )); echo 'Sleep $i.'; sleep 3; done"]
```
2.2 Commit and push the result on your Git fork.

2.3 Rebuild the app in Openshift
```
$ oc start-build sleep
build.build.openshift.io/sleep-2 started
```
2.4 Check the build logs as above
`$ oc logs -f bc/sleep`

2.5 Verify that a new deployment got triggered by the build
``` $ oc status
In project docker-build on server https://openshift:6443

dc/sleep deploys istag/sleep:latest <-
  bc/sleep docker builds https://github.com/svejk-ciber/openshift-dockerfile-example.git#solution on istag/bash:5.0.11
  deployment #2 deployed 4 minutes ago - 1 pod
  deployment #1 deployed 27 minutes ago
...
```
2.6 Wait for the app pod to be available: `$ oc get pod ...` as above
Note that here are now pods of version 1 and 2 of the builds performed. In the end,
only the newly built image pod of the latest build (2), should be kept running.

2.7 View the application log to observe changes
```
$ oc log sleep-2-2zffm...
...
Sleep 8
Sleep 9
Sleep 10
...
```
2.8 Inspect the image stream for the outpur image as above.
Observe that there are now multiple versions of the image in play of the Docker tag `latest`, 
and that the is currently points to the last built image, by SHA256 hash.


3. Cleanup
3.1 `$ oc delete all -l app=sleep`

3.2 `$ oc get all`
Should indicate that there are no resources left in the project.

3.2 Delete the project 
`$ oc delete project docker-build`
