---
layout: post
title:  "differential deploy to Amazon S3 from BitBucket"
date:   2019-07-07 17:10:27 +0200
categories: bitbucket, pipes, deployment, differential, deploy
---

This is my first post on this personal blog, of many I hope.

Under the idea to automate the deployments of our static files and as a step toward a CI/CD environment, and taking adventage from the fact that we recently move all the static files to AWS S3 buckets, I've decided to give it a try automating our process.

## Current state

Our deployment process (the integration one) has two big elements to deploy: A Salesforce package. And a folder with all the static files that the Salesforce package uses, including front end code.

When this Salesforce package is created the repository is tagged with a *TEST_X* name.

**Currently the process** to deploy statics goes like this:

1. Go to Source Tree.
2. Check the files that changed between current and the last TAG.
3. Copy these files into AWS S3
4. Invalidate Cloudfire cache (Amazon's CDN... sort of)

Once this first big deploy has been done it can happen also that minor changes that only require static deployment get deployed individually. Without the need of generating another package.

## The new apporach

Use Bitbucket pipelines to execute a script with what we just described.

There is already a very [good pipeline for AWS S3](http://gfdg.com) ready to use. The only drawback is that the sync commando used to copy the files to S3 uses a combination of timestamp and size of the files to decide wether do the upload or not. Since in BitBucket the pipe executes from a docker container the files that get clones from GIT always have a more recend timestamp than the ones on S3, meaning all the files get copied no matter what.

The only new thing the script will do will be:

1. Use GIT diff to get the changed files (if we are in a TAG or not changes the behaviour)
2. AWS CLI to send the specific files
3. AWS CLI to invalidate cache programatically

### Simple Bitbucket Pipe

The first step is to create a pipe from where we will execute the script.
I think the [official documentation](https://confluence.atlassian.com/bitbucket/how-to-write-a-pipe-for-bitbucket-pipelines-966051288.html) does quite a good job on this matter. Remember we will use the **Simple Pipe** to keep files and code at a minimum. 

The only part that I missed from the bitbucket docs was that it makes you think it's not possible to access repository defined variables from the simple pipe. This is half true since you can actually pass them when using the pipe in your project's bitbucket-pipelines.yml file.

#### Dockerfile

The instructions leave you with a three file project. Let's start with the Dockerfile.

```dockerfile
FROM atlassian/pipelines-awscli:1.16.185
RUN apk update && apk add ca-certificates && rm -rf /var/cache/apk/*
COPY pipe.sh /
ENTRYPOINT ["/pipe.sh"]
```

The only difference from the provided is the first line

> ``FROM  atlassian/pipelines-awscli:1.16.185``

This provides us with the AWS CLI tools pre installed, so we don't have to do it on building time.

#### Pipe Script

Next, we have the script we will use as a pipe, and it would be the one that contains the GIT commands, the deployment and the cache invalidation.
I based this script heavily in what was exposed in [this article](https://www.lambrospetrou.com/articles/aws-s3-sync-git-status/). Let's have a look.

```bash
#!/usr/bin/env bash
#TODO: Apply colors and errors with commons.sh
set -ex

# mandatory parameters
# path variables should not contain slashes
AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID:?'AWS_ACCESS_KEY_ID variable missing.'}
AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY:?'AWS_SECRET_ACCESS_KEY variable missing.'}
S3_BUCKET=${S3_BUCKET:?'S3_BUCKET variable missing.'}
LOCAL_PATH=${LOCAL_PATH:?'LOCAL_PATH variable missing.'}
TAG_REGEX=${TAG_REGEX:?'TAG_REGEX variable missing.'}
CLOUDFRONT_DISTRIBUTION_ID=${CLOUDFRONT_DISTRIBUTION_ID:?'CLOUDFRONT_DISTRIBUTION_ID variable missing.'}


echo "Starting deployment..."
echo "AWS_ACCESS_KEY_ID is: ${AWS_ACCESS_KEY_ID}"
echo "S3_BUCKET is: ${S3_BUCKET}"
echo "LOCAL_PATH is: ${LOCAL_PATH}"
echo "TAG_REGEX: ${TAG_REGEX}"

# This will give us the last two tags created that comply with the TAG_REGEX parameter
TAGS=()
for i in $( git tag -l --sort=refname "${TAG_REGEX}" | tail -2 ); do
    TAGS+=( "$i" )
done

echo "Checking difference between tags:"
echo "${TAGS[0]}"
echo "${TAGS[1]}"

# We will get the list of files that changed between those two tags but keep only the ones that
# begin with the path we want to deploy
FILES=()
for i in $( git diff ${TAGS[0]} ${TAGS[1]} --name-only | sed -n "s|${LOCAL_PATH}/||p" ); do
    FILES+=( "$i" )
done
echo "Files to be deployed..."
printf '%s\n' "${FILES[@]}"

# Construct the parameters to the AWS CLI for the files to includes
CMDS=()
for i in "${FILES[@]}"; do
    CMDS+=("--include=$i""*")
done
echo "${CMDS[@]}"

# Exclude ALL diles and only includes the previously defined.
# Remove dryrun to deploy to aws s3
# Remember this command won't show any output if the file in the bucket is MORE RECENT than the one in local
echo "${CMDS[@]}" | xargs aws s3 sync ./${LOCAL_PATH}/ s3://${S3_BUCKET}/ --dryrun --delete --exclude "*"

aws cloudfront create-invalidation --distribution-id ${CLOUDFRONT_DISTRIBUTION_ID} --paths /folderToInvalidate/*
```

The paremeters section at the beggining it's more as a reminder of what are we going to have during the script, these variables will come from the configuration when using the pipe. It is worth remind you that all the paths should not contain slashes, this is critical
since AWS S3 doesn't allow dopuble slashes.

We list the files that have change, exclude all files from the ´´aws sync´´ command amd include only the ones that have changed.
We finally invalidate cache of directory on cloudfire to be able to see the changes if needed.

Last but not least, before pushing this file to the repo execute a

```bash
chmod -x pipe.sh
```

To have execution permission once in docker.

#### bitbucket-pipelines.yml

This file configures the upload of your pipe to dockerhub, just don't forget to have available in your repo
the docker hub credential variables.

### The project with the static files

Now we configure the pipeline file for our project (bitbucket repo) to use the new pipe and pass the necessary variables.
My ``bitbucket-pipelines.yml`` file looks like this:

```yaml
# This is a sample build configuration for Other.
# Check our guides at https://confluence.atlassian.com/x/5Q4SMw for more examples.
# Only use spaces to indent your .yml configuration.
# -----
# You can specify a custom docker image from Docker Hub as your build environment.
image: atlassian/default-image:2

pipelines:
  custom:
    deploy-statics-diff-last-tags:
      - step:
          script:
            - pipe: docker://gmoretti/aws-deploy-diff-last-tag-simple-pipe:latest
              variables:
                AWS_ACCESS_KEY_ID: '$AWS_ACCESS_KEY_ID'
                AWS_SECRET_ACCESS_KEY: '$AWS_SECRET_ACCESS_KEY'
                S3_BUCKET: 'aws-static-deploy-test'
                LOCAL_PATH: 'static'
                TAG_REGEX: 'TEST_*'
```

Let's go line by line, first thing we declare this as a ``custom`` step, so to use it we need to go to bitbucket branch and manually click on it, next line is the **name of the step** taht will appear in the list of things that can be executed.

```yaml
custom:
    deploy-statics-diff-last-tags:
```

Next, the url to your pipe in dockerhub.

```yaml
            - pipe: docker://gmoretti/aws-deploy-diff-last-tag-simple-pipe:latest
```

Now, we have our AWS credentials, you could easily hardcode them here, but they are better as repository variables, specially now that bitbucket lets you have different ones per environment.

```yaml
            AWS_ACCESS_KEY_ID: '$AWS_ACCESS_KEY_ID'
            AWS_SECRET_ACCESS_KEY: '$AWS_SECRET_ACCESS_KEY'
```

Next, the target bucket without the protocol `S3://` **nor any slash at the end**. You can add the **target path** to the very specific folder you want the files on.

```yaml
                S3_BUCKET: 'aws-static-deploy-test'
```

Since we are in the root of the project we put the path to the directory we want to deploy. Again, no slashes at the start or end.

```yaml
                LOCAL_PATH: 'static'
```

The last variable will help us match with the TAGS we want to compare, so they need to be somehow numerically secuential.

```yaml
                TAG_REGEX: 'TEST_*'
```

This will match all tags like ``TEST_1``, ``TEST_3``, ``TEST_6``, etc.

### Execution

Now, we only need to **push all the changes** go to **branches** in bitbucket and *execute the pipe*.

## Final thoughts

Alotought to solve this specific case, this pipe would work great, the thing is that there are **several other cases** that have not been covered, like, deploying files not covered by a tag or commiting files between several tags. Althought all these things can be done extending the pipe script, the truth is that **just tagging and deploying all the static files will eliminate the whole problem** and will leave as using the bitbucket *AWS S3 mantained pipe*.

That's what I would take home today.

## References

- [BitBucket Pipe Tutorial](https://confluence.atlassian.com/bitbucket/how-to-write-a-pipe-for-bitbucket-pipelines-966051288.html)

- [AWS S3 sync - only modified files, using git status by Lambros Petrou](https://www.lambrospetrou.com/articles/aws-s3-sync-git-status/)

If you have any thoughts feel free to contact me, **thank you!**
