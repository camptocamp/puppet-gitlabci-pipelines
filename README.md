# GitLab CI Pipelines for Puppet tasks

[![By Camptocamp](https://img.shields.io/badge/by-camptocamp-fb7047.svg)](http://www.camptocamp.com)


This is a collection of GitLab CI Pipeline rules that can be used to automate
the following CI tasks:

* Puppet code linting with [onceover](https://github.com/dylanratcliffe/onceover)
* Puppet code deploying with [r10k](https://github.com/puppetlabs/r10k)
* Puppet code impact analysis with [puppet-catalog-diff](https://github.com/camptocamp/puppet-catalog-diff)
* Puppet code cleanup with [puppet-ghostbuster](https://github.com/camptocamp/puppet-ghostbuster)


It is entirely based on containers and plays well with the [Camptocamp Helm charts for Puppet infrastructure](https://github.com/camptocamp/charts).
