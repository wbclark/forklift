# Development Environments

This covers how to setup and configure a development environment using the Forklift tool suite.

 * [Development Environment Deployment](#development-environment-deployment)
 * [Use Koji Scratch Builds](#koji-scratch-builds)
 * [Test Puppet Module Pull Requests](#test-puppet-module-pull-requests)
 * [Jenkins Job Builder](#jenkins-job-builder-development)
 * [Redmine Development](#redmine-development)
 * [Hammer Development](#hammer-development)
 * [Capsule Development](#capsule-development)
 * [Client Development](#client-development)
 * [Dynflow](#dynflow-development)

## Development Environment Deployment

A Katello development environment can be deployed on CentOS 7. Ensure that you have followed the steps to setup Vagrant and the libvirt plugin. There are a variety of useful development environment options that should or can be set when creating a development box. These options are designed to configure your environment ready to use your own fork, and create pull requests. To create a development box:

1. Copy `vagrant/boxes.d/99-local.yaml.example` to `vagrant/boxes.d/99-local.yaml`. If you already have a `99-local.yaml`, you can copy the entries in `99-local.yaml.example` to your `99-local.yaml`.
2. Now, replace `<my_github_username>` with your github username
3. Fill in any ansible options, examples:
   * `foreman_devel_github_push_ssh`: Force git to push over SSH when HTTPS remote is configured
   * `ssh_forward_agent`: Forward local SSH keys to the box via ssh-agent
   * `katello_devel_github_username`: Your GitHub username to set up repository forks
4. Fill in any foreman-installer options, examples:
   * `--katello-devel-use-ssh-fork`: will add your fork by SSH instead of HTTPS
   * `--katello-devel-fork-remote-name`: will change the naming convention for your fork's remote
   * `--katello-devel-upstream-remote-name`: will change the naming convention for the upstream (non-fork) repositories remote
   * `--katello-devel-extra-plugins`: specify other plugins to have setup and configured

For example, if I wanted my upstream remotes to be origin and to install the remote execution and discovery plugins:

```yaml
centos7-katello-devel:
  box: centos7
  ansible:
    playbook: 'playbooks/katello_devel.yml'
    group: 'devel'
    variables:
      katello_devel_github_username: <REPLACE ME>
      foreman_installer_options:
        - "--katello-devel-extra-plugins theforeman/foreman_remote_execution"
        - "--katello-devel-extra-plugins theforeman/foreman_discovery"
```

Lastly, spin up the box:

```
vagrant up centos7-katello-devel
```

The box can now be accessed via ssh and the Rails server started directly (this assumes you are connecting as the default `vagrant` user):

```sh
vagrant ssh centos7-katello-devel
cd foreman
bundle exec foreman start
```

### Starting the Development Server

Our backend requires a rails server to be running. We also use a webpack server for the more "modern" parts of our front-end UI.
[Webpack](https://webpack.js.org/) is a way of bundling up multiple front-end files and assets to a small amount of files to send to the browser.
The files that webpack handles are located in `webpack/` directory found in Foreman, Katello,
and plugin root directories. If you are editing any files in `webpack/` and want to have your changes refresh automatically, you will need a webpack server running.

Because we are using a webpack server in conjunction with a rails server, there are different ways of starting a server depending on your needs and preferences. The following are instructions for starting the server using a base `centos7-katello-devel` box as a starting point.

#### Run a rails and webpack server together using `foreman start`
- Run `bundle exec foreman start` in `~/foreman`
- Navigate to `https://centos7-katello-devel.<hostname>.example.com/` where `<hostname>` is the shortname of your hypervisor (machine your VM is running on).
- Accept the self-signed certs at `https://centos7-katello-devel.<hostname>.example.com:3808`.
- Everything should be set for you to run `bundle exec foreman start` to start your dev server as needed.

NOTE: The `foreman` in `foreman start` is actually [this gem](https://github.com/ddollar/foreman) and not our `foreman`. It
doesn't allow STDIN, which means it doesn't support local debugging. To use a debugger with `foreman start`, you can use
remote debugging with a tool like [pry-remote](https://github.com/Mon-Ouie/pry-remote). There is a helpful blog post about
remote debugging [here](http://blog.honeybadger.io/remote-debugging-with-byebug-rails-and-pow/) with more information.

#### Run the rails server without a webpack server running
If you don't need to do any development in the `webpack/` directories, you can turn the webpack server off and run only the rails server if desired.

- Run `foreman-installer --katello-devel-webpack-dev-server false` to disable the webpack server. You can also set this installer option
in your boxes.yaml entry to configure this on box creation.
- Start the rails server with `bundle exec foreman start rails` in `~/foreman`
  - Alternatively, you can start the rails server with `bundle exec rails s`

#### Additional info
- `foreman start` can be run with `rails` or `webpack` to start just that server: i.e. `bundle exec foreman start rails`. This can be used to start servers separately.

### Reviewing Pull Requests

It's easy to checkout pull requests from projects that were installed in development environment. All projects are cloned in vagrant's home, e.g. ~/foreman, ~/katello etc. In order to apply some PR to any of this project, you can use a reviewer script. See following example

```sh
cd ~/katello
rpr 5266
```

`rpr` is shortcut for review pull request. This fetches information about PR number 5266, defines new remote in you .git/config, fetches new objects from it. Then it creates new local branch called review/pr5266 and pulls the code from the PR. It uses SSH connection so using ssh-agent, with SSH key uploaded to your github account, is recommended. If you're applying a PR on remote machine to which you connected through SSH, make sure you open that connection with agent forwarding, e.g. `ssh -a vagrant@host.example.com`. Don't forget to migrate and seed the database if the PR contains related changes. Note that this might create new merge commit so git might want you to set your email and name. You can set that with following commands.

```sh
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

Once reviewing is finished, the repository can be reset to develop/master branch by calling `rrpr`. It destroys the review branch after it checkouts back to master branch.

If `rpr` is used in project with `config/database.yml` it will also create a backup of the db in ./tmp/. When `rrpr` is called later and in case previous backup was found, it asks whether it should be restored.

## Koji Scratch Builds

Forklift supports using Koji scratch builds to make RPMs available for testing purposes. For example, if you want to test a change to nightly, with a scratch build of rubygem-katello. This is done by fetching the scratch builds, and deploying a local yum repo to the box you are deploying on. Multiple scratch builds are also supported for testing changes to multiple components at once (e.g. the installer and the rubygem), see examples below. Also, this option may be specified from within `99-local.yaml` via the `options:` option.


An Ansible role is provided that can setup and configure a Koji scratch build for testing. If you had an existing playbook such as:

```yaml
- hosts: server
  roles:
    - etc_hosts
    - foreman_repositories
    - katello_repositories
    - katello
```

The Koji role and task ID variable can be added to download and configure a repository with priority:

```yaml
- hosts: server
  vars:
    koji_task_ids:
      - 321231
  roles:
    - etc_hosts
    - koji
    - foreman_repositories
    - katello_repositories
    - katello
```

## Test Puppet Module

### Pull Requests

Testing installer puppet module pull requests is possible through an Ansible variable. Any number of modules and associated pull requests may be specified. For example, if a module under goes a refactoring, and you want to test that it continues to work with the installer. The pull requests are indicated by the github project, repository, and pull request number (eg. katello/qpid/23). Note that the name in this situation is the name as laid down in the module directory as opposed to the github repository name. In other words, use 'qpid' not 'puppet-qpid'. The pull requests are specified through the 'foreman_installer_module_prs' variable in the 'ansible' 'variables' section of your box definition. See examples below.

Single module PR in `99-local.yaml`:
```yaml
ansible:
  variables:
    foreman_installer_module_prs:
      - katello/katello_devel/97
```

Multiple modules:
```yaml
ansible:
  variables:
    foreman_installer_module_prs:
      - katello/katello_devel/97
      - katello/qpid/34
```

### Branches

Testing installer puppet module branches is possible through an Ansible variable. Any number of modules and associated branches may be specified. For example, if a module under goes a refactoring, and you want to test that it continues to work with the installer. The branches are indicated by the github project, repository, and branch (or commit) name (eg. myfork/qpid/my-branch). Note that the name in this situation is the name as laid down in the module directory as opposed to the github repository name. In other words, use 'qpid' not 'puppet-qpid'. The branches are specified through the 'foreman_installer_module_branches' variable in the 'ansible' 'variables' section of your box definition. See examples below.

Single module branch in `99-local.yaml`:
```yaml
ansible:
  variables:
    foreman_installer_module_branches:
      - myfork/katello_devel/switch-to-scl
```

Multiple modules:
```yaml
ansible:
  variables:
    foreman_installer_module_branches:
      - myfork/katello_devel/switch-to-scl
      - myfork/foreman/add-puma
```

## Jenkins Job Builder Development

When modifying or creating new Jenkins jobs, it's helpful to generate the XML file to compare to the one Jenkins has. In order to do this, you need a properly configured Jenkins Job Builder environment. The dockerfile under docker/jjb can be used as a properly configured environment. To begin, copy `docker-compose.yml.example` to `docker-compose.yml`:

```
cd docker/jjb
cp docker-compose.yml.example docker-compose.yml
```

Now edit the docker-compose configuration file to point at your local copy of the `foreman-infra` repository so that it will mount and record changes locally when working within the container. Ensure that either your docker has permissions to the repository being mounted or that the appropriate Docker SELinux context is set: [Docker SELinux with Volumes](http://www.projectatomic.io/blog/2015/06/using-volumes-with-docker-can-cause-problems-with-selinux/). Now we are ready to do any Jenkins Job Builder work. For example, if you wanted to generate all the XML files for all jobs:

```
docker-compose run jjb bash
cd foreman-infra/puppet/modules/jenkins_job_builder/files/theforeman.org
jenkins-jobs -l debug test -r . -o /tmp/jobs
```

## Redmine Development

The Foreman project uses Redmine to handle issue management via forked instance of Redmine that runs on Openshift. Testing upgrades, making plugins or patches is sometimes desired to achieve functionality which we need. The dockerfile under docker/redmine can be used as a properly configured Redmine environment for development. To begin, copy `docker-compose.yml.example` to `docker-compose.yml`:

```
cd docker/redmine
cp docker-compose.yml.example docker-compose.yml
```

Assuming you have a clone of the Redmine repository somewhere locally, edit the `docker-compose.yml` configuration file to point at your local copy of the `redmine` repository so that it will mount and record changes locally when working within the container. Ensure that either your docker has permissions to the repository being mounted or that the appropriate Docker SELinux context is set: [Docker SELinux with Volumes](http://www.projectatomic.io/blog/2015/06/using-volumes-with-docker-can-cause-problems-with-selinux/). Now we are ready to start up Redmine and make changes:

```
docker-compose up redmine
```

## Hammer Development

Hammer is the command line interface (CLI) to Foreman and Katello. It supports plugins
such as [Foreman Tasks](https://github.com/theforeman/hammer-cli-foreman-tasks) and
importing/exporting data via [CSV](https://github.com/Katello/hammer-cli-csv).
The CLI can be configured to work with any version of Foreman. To facilitate
development in Hammer or any of its plugins, a lightweight vagrant box is
provided in the `boxes.yaml.example` file. To use this functionality, copy the
centos7-hammer-devel configuration from the example file into your `boxes.yaml`
file, changing options as necessary. Then run the following:

```sh
vagrant up centos7-hammer-devel
```

In the vagrant box, find the Hammer repositories at `/home/vagrant/` and the
configuration at `/home/vagrant/.hammer`.

## Capsule Development

To use this functionality, add the following configuration to your `99-local.yaml`,
changing the hostnames as needed

### To setup a smart proxy (capsule) and a new development environment

* setup 99-local.yaml

```yaml
capsule-dev:
  box: centos7
  ansible:
    playbook: 'playbooks/foreman_proxy_content_dev.yml'
    group: 'foreman-proxy-content'
    server: 'foo'
```
* ```vagrant up foo```
* ssh into foo and ```rails s```
* ```vagrant up capsule-dev```


### To setup a capsule with an existing development environment

* Add the following to the existing Katello development server's configuration in `99-local.yaml`
```yaml
  ansible:
    group: 'server'
```
* Add a box for a capsule, using the katello server's name in the "server" field:

```yaml
capsule-dev:
  box: centos7
  ansible:
    playbook: 'playbooks/foreman_proxy_content_dev.yml'
    group: 'foreman-proxy-content'
    server: 'your-katello-server-name'
```
* ssh into existing development server and ```rails s```
* spin up new capsule ```vagrant up capsule-dev```

## Client Development

`99-local.yaml` defines a `katello-client` box which can be used to register against a Katello instance.
The following example shows some of the extra values that can be set to control how the client is registered.

```yaml
katello-client:
  box: centos7
  ansible:
    playbook: 'playbooks/katello_client.yml'
    group: 'client'
    variables:
      katello_client_server: 'centos7-katello-devel'
      katello_client_organization: 'Default_Organization'
      katello_client_environment: 'Library'
      katello_client_username: 'admin'
      katello_client_password: 'changeme'
      katello_client_install_agent: True
```

## Dynflow Development

DYNamic workFLOW orchestration engine http://dynflow.github.io

To use this box, copy the configuration from `boxes.yaml.example` to
`boxes.yaml`, changing options as necessary, then run the following:

```sh
vagrant up centos7-dynflow-devel
```

In the vagrant box, the dynflow repository is cloned to `/home/vagrant/dynflow`.

## Pulp3 deployment

To deploy pulp3 onto an existing box, within forklift run:
```
git clone https://github.com/pulp/ansible-pulp.git
cd ansible-pulp
ansible-galaxy install -r requirements.yml -p ./roles
cd ..
```
