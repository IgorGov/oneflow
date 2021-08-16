# Oneflow implementation using Github action

## Introduction
This repo is an example of an [Oneflow](https://www.endoflineblog.com/oneflow-a-git-branching-model-and-workflow) 
implementation using Github acttions and deployment to [GKE](https://cloud.google.com/kubernetes-engine). 

## Background
At somepoint while working for [UP9](https://github.com/up9inc), I started wondering whether we are using the right 
Git Workflow. Back then we worked with the well known [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/).

What was the trigger to check the alternatives?
* Promotions to production took too much time. We have different repo for each microservice, some of them have a long 
  build time.
* The merges to master usually had conflicts and were complicated task with no reason.
* Once build of one of the microservices failed after merge to master, we ended up with different versions of 
  microservices in prod. 

It raised one question, if we're already building an image for each development push, testing it in staging environment 
why can't we just use the same image for production?

## Using Oneflow as the main Git Workflow
The theory about Oneflow can be found [here](https://www.endoflineblog.com/oneflow-a-git-branching-model-and-workflow).
I will focus here mainly on implementation. Let's assume that we're using "develop" as our main branch, all other 
branches are short living like "feature/abc", "bug/xyz", or "hotfix/v1.0.0". <br />
We have 3 main workflows
* Adding features and fixing bugs on development
* Promotion to production of a new "stable version"
* Hotfix to production

### Ongoing work
After we're done implementing new feature or fixing a bug, and we're pushing the changes to develop, a Github action 
triggered [build-publish-deploy](.github/workflows/build-publish-deploy.yaml):
* build a docker image
* publish to container registry [GCR](https://cloud.google.com/container-registry)
* add docker tags:
  * gcr.io/\<project id>/\<git repo name>/develop:latest
  * gcr.io/\<project id>/\<git repo name>/develop:\<commit sha>
  * gcr.io/\<project id>/\<git repo name>/release-candidate:\<commit sha>
  
* Rollout restart k8s deployment in staging 
  * the node will be restarted and take the new image from: gcr.io/\<project id>/\<git repo name>/develop:latest

### Promotion to prod
Once an image passed all the tests, and we sure that it stable
* mark the commit with git tag
  * `git tag v1.0.0`
  
* Push the tags to remote
  * `git push --tags`
  
A Github action triggered [tag-stable](.github/workflows/tag-stable.yaml), which does only one thing:
* extracts the commit sha from the tag 
  
* adds tag to the image in .../release-candidate:\<commit sha> with .../stable:latest
* rollout restart k8s deployment in prod

### Hotfix
Once in a while there is a bug in production, and we can't promote the latest development, so we need a hotfix. <br />
In this case we should 
* create a hotfix branch from the tag of the last release: <br />
`git checkout -b hotfix/some_bug v1.0.0` <br />
  
* push it to remote <br />
`git push -u origin hotfix/some_bug`<br/>
* [build-publish-deploy](.github/workflows/build-publish-deploy.yaml) is triggered, will build & publish an image 
  (without deploying it to staging)
  
* test the image in test env

Once the hotfix passed all the tests, 
* mark the commit with git tag
  * `git tag v1.0.1`
  
* Push the tags to remote
  * `git push --tags`
  
* [tag-stable](.github/workflows/tag-stable.yaml) triggered, will add tag of stable:latest to image and deploy to prod

Don't forget to merge the 'hotfix/some_bug' to develop, so the new tag v1.0.1 won't be lost, if you remove the branch.


