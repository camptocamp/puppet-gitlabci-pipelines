# GitLab CI Pipelines for Puppet tasks

[![By Camptocamp](https://img.shields.io/badge/by-camptocamp-fb7047.svg)](http://www.camptocamp.com)


This is a collection of GitLab CI Pipeline rules that can be used to automate
the following CI tasks:

* Puppet code linting with [onceover](https://github.com/dylanratcliffe/onceover)
* Puppet code deploying with [r10k](https://github.com/puppetlabs/r10k)
* Puppet code impact analysis with [puppet-catalog-diff](https://github.com/camptocamp/puppet-catalog-diff)
* Puppet code cleanup with [puppet-ghostbuster](https://github.com/camptocamp/puppet-ghostbuster)


It is entirely based on containers and plays well with the [Camptocamp Helm charts for Puppet infrastructure](https://github.com/camptocamp/charts).


## How to use

In your `.gitlab-ci.yml`, include the library file, then extend the rules, e.g.:

```yaml
---
stages:
  - lint
  - deploy
  - diff
  - ghostbuster
  - purge

include:
  - 'https://raw.githubusercontent.com/camptocamp/puppet-gitlabci-pipelines/master/.gitlab-ci.yml'
```

## Rules


### Onceover

The `.onceover` rule runs linting tasks on the Puppet code.

Example usage:

```yaml
linting-puppet-hiera:
  extends: .onceover
  stage: lint
  rules:
    - if: '$CI_MERGE_REQUEST_ID'
```


### R10K

There are two R10k-related rules:

- `.r10k-deploy` for the environment deployment
- `.r10k-purge` for environment purging

These rules are meant to be used with GitLab Environments to manage the
lifecycle of Puppet environments individually:

```yaml
r10k-deploy:
  extends: .r10k-deploy
  stage: deploy
  variables:
    R10K_BRANCH: '$CI_COMMIT_REF_NAME'
    R10K_ENV: '$CI_COMMIT_REF_NAME'
  environment:
    name: 'environment/${CI_COMMIT_REF_NAME}'
    on_stop: r10k-purge
  rules:
    - if: '$CI_MERGE_REQUEST_ID'
      when: never
    - if: '$GHOSTBUSTER != "true"'
  resource_group: 'r10k/${CI_COMMIT_REF_NAME}'

r10k-purge:
  extends: .r10k-purge
  stage: purge
  variables:
    R10K_ENV: '$CI_COMMIT_REF_NAME'
  environment:
    name: 'environment/${CI_COMMIT_REF_NAME}'
    action: stop
  rules:
    - if: '$CI_MERGE_REQUEST_ID'
      when: never
    - if: '$CI_COMMIT_REF_NAME =~ /^sta(ble|ging)/'
      when: never
    - when: manual
```


### G10K

If you prefer to use G10K instead of R10K, extend the `.g10k-deploy` rule
instead of `.r10k-deploy`.



#### Requirements

- A GitLab runner mounting (RW) the Puppet code inside `/etc/puppetlabs/puppet/code`
  on its executors


#### Variables

The rules take two variables:

- `$R10K_BRANCH`: the branch to checkout, useful when checking out a dynamic
  branch (e.g. from a Merge Request for catalog diff)
- `$R10K_ENV`: the target branch used to name the environment


### Catalog Diff

The `.catalog-diff` rule is meant to be used in a Merge Request to provide
impact analysis of the merge.

We recommend to set it up by deploying an additional environment which results
of the merge, and running Catalog Diff on it:

```yaml
r10k-deploy-mr:
  extends: .r10k-deploy
  stage: deploy
  variables:
    R10K_BRANCH: '$CI_MERGE_REQUEST_REF_PATH'
    R10K_ENV: 'mr_${CI_MERGE_REQUEST_IID}'
  environment:
    name: 'merge/${CI_MERGE_REQUEST_IID}'
    on_stop: r10k-purge-mr
  rules:
    - if: '$CI_MERGE_REQUEST_ID && $CI_COMMIT_MESSAGE =~ /\[autodiff.*\]/'
    - if: '$CI_MERGE_REQUEST_TITLE =~ /\[autodiff.*\]/'
    - if: '$CI_MERGE_REQUEST_ID'
      when: manual
  resource_group: 'r10k/mr_${CI_MERGE_REQUEST_IID}'
  allow_failure: true

r10k-purge-mr:
  extends: .r10k-purge
  stage: purge
  variables:
    R10K_ENV: 'mr_${CI_MERGE_REQUEST_IID}'
  environment:
    name: 'merge/${CI_MERGE_REQUEST_IID}'
    action: stop
  rules:
    - if: '$CI_MERGE_REQUEST_IID'
      when: manual

catalog-diff:
  extends: .catalog-diff
  stage: diff
  environment:
    name: 'diff/${CI_MERGE_REQUEST_IID}'
    url: '${PUPPETDIFF_URL}/?report=mr_${CI_MERGE_REQUEST_IID}_${CI_JOB_ID}'
  variables:
    DIFF_FLAGS: --show_resource_diff --exclude_classes --exclude_defined_resources --changed_depth 1000 --old_catalog_from_puppetdb --certless --threads 8 --ignore_parameters alias --exclude_resource_types Stage
    REPORT: mr_${CI_MERGE_REQUEST_IID}_${CI_JOB_ID}
    PUPPETDIFF_URL: https://puppetdiff.example.com
  rules:
    - if: '$CI_MERGE_REQUEST_ID && $CI_COMMIT_MESSAGE =~ /\[autodiff\]/'
    - if: '$CI_MERGE_REQUEST_TITLE =~ /\[autodiff\]/'
    - if: '$CI_MERGE_REQUEST_ID'
      when: manual

catalog-diff-purge:
  image: busybox
  stage: purge
  environment:
    name: 'diff/${CI_MERGE_REQUEST_IID}'
    action: stop
  variables:
    GIT_STRATEGY: none
  script:
    - rm -rf /catalog-diff/mr_${CI_MERGE_REQUEST_IID}_*.json
  allow_failure: true
```


#### Requirements

- A GitLab runner mounting (RO) the Puppet code inside `/etc/puppetlabs/puppet/code`
- The `camptocamp-catalog_diff` installed in the source branch (for example
  using a `Puppetfile` with r10k)
- If you want to store reports permanently, a GitLab runner mounting (RW)
  a volume on the `/catalog-diff` directory (which can then be used with
  puppet-catalog-diff-viewer for example)


#### Variables

The rule uses three variables:

- `$DIFF_FLAGS`, the flags passed to catalog-diff
- `$REPORT`, the name of the report written on disk
- `$PUPPETDIFF_URL`, the URL posted at the end of the run, typically pointing
  to the location of a [`puppet-catalog-diff-viewer`](https://github.com/camptocamp/puppet-catalog-diff-viewer) instance where the report can be inspected
- `$CI_BOT_TOKEN`, the GitLab token used to post the diff URL to the Merge
  Request as a comment


### Ghostbuster


The `.ghostbuster` rule is meant to be launched as a cron job on the branch of
your choice, to find dead code in it:

```yaml
ghostbuster:
  extends: .ghostbuster
  stage: ghostbuster
  variables:
    HIERA_EYAML_PATH: ./hiera.yaml
  rules:
    - if: $GHOSTBUSTER == "true"
```


#### Requirements

- A `Rakefile` in the current directory with a `ghostbuster` task.


#### Variables

In the example above, the `$GHOSTBUSTER` variable is used to limit the scope of the run.
This variable is then set to `"true"` in the cron job to launch this rule.
