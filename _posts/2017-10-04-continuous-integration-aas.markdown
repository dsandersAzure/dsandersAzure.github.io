---
layout: post
title: Continuous Integration as a Service
date: 2017-09-28 13:24:55 +0000
categories:
- Continuous Integration
- SaaS
- PetProject
---
A solution that I've become really enamoured with is [CircleCI](https://circleci.com/) which offers SaaS and on-premises tools for continuous integration.  What I really like is a simple YAML configuration controls the build pipeline on every commit.

**tl&dr : Use simple yaml configuration files with CircleCI.com to automatically prepare, pull, test, and deploy to Docker and Google App Engine.** 

Quick notes:
* The [yaml configuration file](https://github.com/dsandersAzure/python_cowbull_server/blob/master/.circleci/config.yml) is in my repository and publicly available.
* The CircleCI build will **not** complete if the Google Cloud app is disabled (a state I often use to save money).
* References and attributions are at the end of this post.

My yaml configuration file consists of a number of sections (*NB* do **not** copy and paste from here due to formatting issues; please use the repo [file](https://github.com/dsandersAzure/python_cowbull_server/blob/master/.circleci/config.yml)).

#### Step 1, Spin up the environment

{% highlight yaml %}
{% raw %}
build:
  docker:
    - image: circleci/python:3.6.1
    - image: redis
  working_directory: ~/python_cowbull_server
  environment:
    - REDIS_PORT: 6379
    - REDIS_HOST: localhost
{% endraw %}
{% endhighlight %}


This simple script stands up two Docker containers; the first provides the Python 3.6 runtime area, the second a Redis cache. The env. vars. simply tell my app where to find Redis.

#### Step 2, Restore the cache and install the dependencies

{% highlight yaml %}
{% raw %}
steps:
- checkout
- restore_cache:
    keys:
      - v2-dependencies-{{checksum "requirements.txt"}}
      - v2-dependencies-
  - run:
      name: install dependencies
      command: |
        python3 -m venv venv
        . venv/bin/activate
        pip install -r requirements.txt
{% endraw %}
{% endhighlight %}

All that's happening here is we are restoring our environment dependencies cache (if it exists or defaulting if it doesn't) and then setting up a Python virtualenv and installing the project's requirements. We could easily add more virtual environments, e.g. for Python 2.7., simply by repeating these steps.

#### Step 3 Run the tests

{% highlight yaml %}
{% raw %}
- run:
    name: run tests
    command: |
      . venv/bin/activate
      set -o pipefail
      python -m unittest -v tests 2> >(tee -a /tmp/unittest-report.log >&2)
{% endraw %}
{% endhighlight %}

There's an interesting piece here. Notice that I've set pipefail to ensure that test failure is detected, and I've also redirected the error output to a file using process redirection with tee. There's a few reasons for this but I'd like to direct you back to the [original StackOverflow question](https://stackoverflow.com/questions/692000/how-do-i-write-stderr-to-a-file-while-using-tee-with-a-pipe) (1) that does clever stuff with process redirection and resolved my issues - it's worth reading.

#### Step 4 Install Google Cloud dependencies

{% highlight yaml %}
{% raw %}
- run:
    name: install Google Cloud dependencies
    command: |
      # Fix the repo to Jessie - needs to be updated later
      export CLOUDSDK_CORE_DISABLE_PROMPTS=1
      export CLOUD_SDK_REPO="cloud-sdk-jessie"
      echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
      curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo apt-get update && sudo apt-get install google-cloud-sdk --yes
      echo $circlecibuilder | base64 --decode --ignore-garbage > auth.json
      gcloud auth activate-service-account $circleciuser --key-file auth.json --project=$gproject
{% endraw %}
{% endhighlight %}

This section basically downloads and installs the Google gcloud environment and uses env. vars. (defined in the CircleCI console) to provide the credentials to a minimum privileged service account that is able to deploy to Google App Engine.

The steps for this were heavily influenced by a really good [Atlassian](www.atlassian.com) (2) article on [deploying to Google](https://confluence.atlassian.com/bitbucket/deploy-to-google-cloud-900820342.html) using BitBucket pipelines. I borrowed these steps for a CircleCI approach.

One final note, the project is set in an env. var. so it can be easily changed.

#### Step 5 Build the GAE environment and deploy

{% highlight yaml %}
{% raw %}
- run:
    name: Build GAE project and deploy
    command: |
      export CLOUDSDK_CORE_DISABLE_PROMPTS=1
      mkdir -p vendor/gae
      pip install -t vendor/gae/lib -r requirements.txt
      gcloud app deploy --version="cigen-"$MAJOR_VERSION"-"$MINOR_VERSION"-"$CIRCLE_BUILD_NUM --quiet
{% endraw %}
{% endhighlight %}

Here, I've disabled prompts and then created a vendor structure for my GAE project to contain a library of my python packages required to run the app. *Notice* that the path *isn't* in the repo, because I don't want a host of Python libs in my git repository.

When the requirements have been installed I  app deploy setting the version name to MAJOR-MINOR-BUILD_NUM. I could do more here and set the traffic split, etc., but don't for testing purposes.

#### Step 6 Build and push the Docker image

{% highlight yaml %}
{% raw %}
- run:
    name: build docker image
    command: |
      docker login --username=$doclog --password=$docpass
      docker build -f vendor/docker/Dockerfile -t dsanderscan/$IMAGE_NAME:$MAJOR_VERSION"."$MINOR_VERSION"."$CIRCLE_BUILD_NUM .
- run:
    name: push docker image
    command: |
      docker tag dsanderscan/$IMAGE_NAME:$MAJOR_VERSION"."$MINOR_VERSION"."$CIRCLE_BUILD_NUM dsanderscan/$IMAGE_NAME:latest
      docker push dsanderscan/$IMAGE_NAME:$MAJOR_VERSION"."$MINOR_VERSION"."$CIRCLE_BUILD_NUM
      docker push dsanderscan/$IMAGE_NAME:latest
{% endraw %}
{% endhighlight %}

The Docker image is built (note env. vars. are again used for credentials) using the dockerfile contained in the repo (in the vendor structures) and then tagged with the build number and latest. If there was a pre-release stage, the latest tag could be left until after that stage was completed.

#### Step 7 Finalize and tidy-up

{% highlight yaml %}
{% raw %}
- save_cache:
    key: v2-dependencies-{{ checksum "requirements.txt" }}
    paths:
      - ~/.m2
- store_artifacts:
    path: /tmp/unittest-report.log
    destination: unittest-report
{% endraw %}
{% endhighlight %}

And that's it. The final steps are to save the cache and then upload the test output to the artifact repository.

Done in about 5 mins as soon as a commit is pushed!

Enjoy the work from the CircleCI team in San Francisco.

[https://circleci.com/](https://circleci.com)

### References
1. *How do I write stderr to a file while using "tee" with a pipe?* (2009). Available at: [https://stackoverflow.com/questions/692000/how-do-i-write-stderr-to-a-file-while-using-tee-with-a-pipe](https://stackoverflow.com/questions/692000/how-do-i-write-stderr-to-a-file-while-using-tee-with-a-pipe) (Accessed: 04 October 2017)

2. Atlassian (2017) *Deploy to Google Cloud*. Available at: [https://confluence.atlassian.com/bitbucket/deploy-to-google-cloud-900820342.html](https://confluence.atlassian.com/bitbucket/deploy-to-google-cloud-900820342.html) (Accessed: 04 October 2017)
