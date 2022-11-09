<h1 align="center">GitLab CI for Rails </h1>

![](gitlab.jpg)

GitLab CI/CD is GitLab’s built-in tool for software development using continuous methodology:

Continuous integration (CI).
Continuous delivery and deployment (CD).


[Gitlab Documentation](https://docs.gitlab.com/ee/ci)

## Configure the project
GitLab CI/CD pipelines are configured using a YAML file called .gitlab-ci.yml within each project. The .gitlab-ci.yml file defines the structure and order of the pipelines and determines:
This is what the .gitlab-ci.yml file looks like for this project:

### Creating a .gitlab-ci.yml file

The .gitlab-ci.yml file is where you configure what CI does with your project. It lives in the root of your repository.

On any push to your repository, GitLab will look for the .gitlab-ci.yml file and start jobs on Runners according to the contents of the file, for that commit.

```yaml
#docker build for ci running
image: gitlab.codica.com:1234/project/project-image:build-1.4

#steps to execute our CI
stages:
  - build 
  # for running build 
  - linters
  #for running code quality tools (rubocop, slim-lint)
  - tests
  #for running rspec
  - scanners 
  # for scan our app adn images on vulnerabilities
  - deploy
  #for deploy your app to areas (staging, production)

#define what docker containers should be linked with your base image
services:
  - postgres:latest
  - redis:latest

# when developing software depends on other libraries which are fetched via the internet during build time it’s shared between pipelines and jobs

variables:
  POSTGRES_DB: test
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: ""
  POSTGRES_HOST: postgres
  DANGER_GITLAB_API_BASE_URL: https://gitlab.codica.com/api/v4
  RAILS_ENV: test

# Use for do this action after espessial job which use (<<: *registry_auth)
.registry_auth: &registry_auth
  image: public.ecr.aws/o0j8c5i3/codica:registry-auth-leaks
  before_script:
    - aws ecr get-login-password --region $AWS_REGION | docker login -u AWS --password-stdin $AWS_REGISTRY_URL


# We use Rubocop for scann and format our code
rubocop:
  stage: linters
  needs: [Build]
  image: $REGISTRY_IMAGE_ID
  except:
    - master
  script:
    - bundle exec rubocop

# We use for scna our projects on vulnerabilities
brakeman:
  stage: linters
  needs: [Build]
  image: $REGISTRY_IMAGE_ID
  except:
    - master
  script:
    - brakeman
  
  bundle_audit:
  stage: linters
  needs: [Build]
  image: $REGISTRY_IMAGE_ID
  except:
    - master
  script:
    - bundle exec bundle-audit

# Trivy we use for security scan our docker images 
Trivy:
  stage: linters
  allow_failure: true
  image:
    name: aquasec/trivy:0.31.3
    entrypoint: [""]
  needs: [Build]
  except:
    - main
  script:
    - trivy image --security-checks vuln $REGISTRY_IMAGE_ID

# We use GitLeaks for detect maybe some leaks and secrets in our repos(espessialy AWS keys)
Gitleaks:
  <<: *registry_auth
  stage: linters
  allow_failure: true
  needs: []
  except:
    - main
    - master
  script:
    - gitleaks detect --verbose --no-git | jq -r '["Description:", .Description], ["File:", .File], ["Line:", .StartLine, "Column:", .StartColumn, "--------------"]' | tr -d '[],""'

# We use Hadolint for check our Dockerfiles on good syntaxis  
Hadolint:
  needs: []
  image: hadolint/hadolint:latest-debian
  stage: scanners
  allow_failure: false
  script:
    - hadolint Dockerfile
    - hadolint Dockerfile.dev
  rules: 
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never
    - changes: 
        - Dockerfile*

# We use Yamllint for check our *.yaml files syntaxis
Yamllint:
  needs: []
  image:
    name: public.ecr.aws/**/codica:yamllint
    entrypoint: [""]
  stage: scanners
  allow_failure: false
  script:
    - rm -rf spec
    - yamllint .
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never
    - changes:
        - "**/*.yml"
        - "**/*.yaml"


# Use rspec for test our application
rspec:
  stage: tests
  image: $REGISTRY_IMAGE
  needs: [Build | Staging]
  variables:
    DB_HOST: postgres
    DB_USERNAME: postgres
    DB_PASSWORD: ""
    DB_PORT: 5432
    POSTGRES_DB: api_test
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: ""
    POSTGRES_HOST: postgres
    RAILS_ENV: test
    REDIS_URL: "redis://redis:6379"
  services:
    - postgres:10.11
    - redis:6.2.1-alpine
  script:
    - bundle exec rails db:migrate RAILS_ENV=test
    - bundle exec rspec
  except:
    - main
    - master

# Build Application before tests
# We Use Kaniko for build our apps(and don't use some keys in our gitlab-runner which depends on security)

Build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  except: 
    - master
  before_script: 
    - mkdir -p /kaniko/.docker
    - echo "{\"credHelpers\":{\"$AWS_REGISTRY_URL\":\"ecr-login\"}}" > /kaniko/.docker/config.json
  script:
    - /kaniko/executor --destination "${REGISTRY_IMAGE}"  
       --build-arg RAILS_MASTER_KEY=${RAILS_MASTER_KEY}
       --context "${CI_PROJECT_DIR}"
       --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
```


### Push .gitlab-ci.yml to GitLab

Once you’ve created .gitlab-ci.yml, you should add it to your Git repository and push it to GitLab.

```
git add .gitlab-ci.yml
git commit -m "Add .gitlab-ci.yml"
git push origin master
```

### Configuring a Runner

In GitLab, Runners run the jobs that you define in .gitlab-ci.yml. A Runner can be a virtual machine, a VPS, a bare-metal machine, a docker container or even a cluster of containers. GitLab and the Runners communicate through an API, so the only requirement is that the Runner’s machine has network access to the GitLab server.

A Runner can be specific to a certain project or serve multiple projects in GitLab. If it serves all projects it’s called a Shared Runner.

Find more information about different [Runners](https://docs.gitlab.com/ee/ci/runners/README.html) in the Runners documentation.

## Regular pipeline graphs
Regular pipeline graphs show the names of the jobs of each stage. Regular pipeline graphs can be found when you are on a single pipeline page. For example:
![pipline](gitlab-ci.png)

## Status of your pipeline and jobs

After configuring the Runner successfully, you should see the status of your last commit change from pending to either running, success or failed.

![status](job-status.png)

By clicking on a job’s status, you will be able to see the log of that job. This is important to diagnose why a job failed or acted differently than you expected.

![error-log](job-log.png)

## Examples

Visit the [examples README](https://docs.gitlab.com/ee/ci/examples/README.html) to see a list of examples using GitLab CI with various languages.



## License
Copyright © 2015-2019 Codica. It is released under the [MIT License](https://opensource.org/licenses/MIT).

## About Codica

[![Codica logo](https://www.codica.com/assets/images/logo/logo.svg)](https://www.codica.com)

The names and logos for Codica are trademarks of Codica.

We love open source software! See [our other projects](https://github.com/codica2) or [hire us](https://www.codica.com/) to design, develop, and grow your product.