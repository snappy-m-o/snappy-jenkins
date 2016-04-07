[![Build Status][travis-image]][travis-url]
# Snappy Jenkins CI

This repo helps setting up the CI environment used for executing Snappy integration tests [1]. It contains Dockerfiles which define the required containers (Jenkins master and slaves and a Nginx proxy) and several shell scripts for bringing them to live both in a cloud provider (only OpenStack supported at the moment) and locally.

The testing jenkins jobs use the binary from this project [2] for launching the tests, it takes care of determining the most recent Snappy cloud image for a given channel and release (see below for the required name patterns), instantiating a cloud instance of that image and getting its ip, executing the integration suite in the instance and shutting it down when finished or in case of errors.

There are also jobs configured for accessing the Snappy Product Integration environment. This gives a lot more flexibility when executing the tests not being only limited to cloud instances, however you must provide credentials for this service.

## Provision

There are provision scripts for creating the CI environment locally and in the cloud. These scripts use docker-machine and docker-compose, so you need to have them installed, see [https://docs.docker.com/machine/install-machine/](here) and [https://docs.docker.com/compose/install/](here). Once the provision is finished you can use the common commands of machine and compose, for instance, to list the machines:

    docker-machine ls

To point your docker client to the engine in a local provisioned server:

    eval $(docker-machine env snappy-jenkins-local)

The same for the cloud environment:

    eval $(docker-machine env snappy-jenkins-remote)

Once that's done you can use your local docker client as if you were in the remote machine, try `docker info`, `docker images`, `docker ps` and the like.

With the client configured to connect to the remote host you can also use compose:

    docker-compose -f ./config/compose/cluster.yml down
    docker-compose -f ./config/compose/cluster.yml pull
    docker-compose -f ./config/compose/cluster.yml up -d

More on [https://docs.docker.com/machine/](docker-machine) and [https://docs.docker.com/compose/](docker-compose) at their respective project pages.

### Local Provision

We have verified the installation and configured the provision script for using the docker-machine kvm driver, it can be installed following the instructions at [https://github.com/dhiltgen/docker-machine-kvm](the project page).

You can setup the enviroment locally by executing this command:

    $ ./bin/jenkins/local-provision.sh

This command creates three kinds of containers:

* Jenkins master instance: the actual Jenkins server, the jobs, plugins and configuration options are stored here.

* Jenkins slave instances: the jobs are executed in this containers. We have instances of different distributions so that we can get tests results from each of them.

* Reverse proxy instance: it has an Nginx process listening on port 8081 that forwards requests to path ```/ghprbhook``` to the jenkins master instance. This is useful only for the cloud deployment, see below

Once the scripts finish you can check the machine ip executing `docker-machine ip snappy-jenkins-local` and access the jenkins master instance from the browser at ```http://<server_ip>:8080```

If you want to try the jobs you can download the credentials from an accessible running server, see `Initializing servers` below.

### Cloud provision

In this case you need to have OpenStack credentials loaded, for example by sourcing a novarc file:

    $ source path/to/openstack/credentials/novarc

Before executing the provision script you should have at least the ```$OS_USERNAME``` and ```$OS_REGION_NAME``` environment variables set. This credentials are used for creating the hosts where the CI environment itself is going to run, and may be different to the ones used later for the Ubuntu Core instances, more on this later. The scripts also assume that you have a valid private key at `~/.canonistack/${OS_USERNAME}_${OS_REGION_NAME}.key`, if that's not the case you can pass the path as the last argument.

The cloud provision process relies on the existence of a prebuilt image on the glance endpoint with the required setup in place. This images can be created with the command:

    $ ./bin/jenkins/create-image.sh

This command creates a new seed instance based on trusty64, provisions it with the 1.9.1 version of docker (with the AUFS storage backend enabled) and sets up some additional packages, paths and files required for the system. Then a new image created from it is stored so that it can be used to create new instances.

There are additional requirements for the cloud provision besides having loaded OpenStack credentials. In this case the setup is done in a cloud instance and you should be able to spin it up. The needed packages can be installed in Ubuntu with:

    $ sudo apt-get install cloud-utils python-openstackclient

To setup the CI instance in the cloud just execute:

    $ ./bin/jenkins/cloud-provision.sh

This command creates a new VM with the same kind of containers as in the local provision, but now the proxy container is used for receiving connections from github, where the Snappy project is held, in order to trigger job executions in response to PR submissions. This is described in the following image:

![Block Diagram](/img/snappy-jenkins.png?raw=true)

The provision command also sets up a security group that allows access to port 8081 from everywhere and ports 8080 and 22 from a local range (by default 10.0.0.0/8). All this configuration is done this way in order to facilitate a secure connection from the GitHub webhook, as detailed in the next section.

You can retrieve the job history from an accessible running server, and also initialize the required credentials in order to run jobs, see `Syncing servers` below.

## Redeploy

Once the server is deployed in order to update the containers you can use the redeploy script:

    $ ./bin/jenkins/cloud-redeploy.sh

It stops the running containers, pulls the new images and restarts the cluster.

## GitHub integration

The Jenkins master container has the GitHub Pull Request Builder Plugin installed [3] to allow the triggering of jobs in response to events from GitHub. The default configuration in the container leaves almost all setup, there are only a few things left to do:

* First of all, you should assign a floating IP to your VM instance so that it can be reached from GitHub. The container infrastructure and security group assigned make it secure to expose it to the wild, as explained earlier.

* The ```github-snappy-integration-tests-cloud``` job is configured to receive payloads from GitHub in response to events. It points to the ubuntu-core/snappy repository, you can change this to access one of your own repos to try the hook.

* In the settings page of the repository configured in ```github-snappy-integration-tests-cloud``` you should setup the webhook to notify Jenkins of changes in the repository, the URL should be ```http://<floating_ip>:8081/ghprbhook/``` and the events to trigger the webhook "Pull Request" and "Issue Comment"

* In order to be able to respond to comments in the pull request and update the status of the tests once they are triggered you should setup a bot user to be managed by Jenkins (a regular user works too). This user should be added as a collaborator in the repo and its credentials must be added in the jenkins general configuration (Manage Jenkins -> Configure System -> GitHub Pull Request Builder -> GitHub Auth -> Credentials -> Add).

## Required images

With cloud provision, in order to spin up the jenkins host an image with a name of the form "wily-daily-amd64" should be accessible for the current user.

With both kinds of provision, for the jobs to be able to run your openstack user should be able to access snappy images with a name of the form ```ubuntu-core/custom/ubuntu-rolling-snappy-core-amd64-edge*``` for the rolling release tests and ```ubuntu-core/custom/ubuntu-1504-snappy-core-amd64-edge*``` for the 15.04 release tests. This kind of images can be created with the snappy-cloud-image tool [4]. The configuration can be changed with the release and channel switches in the respective jobs.

## Data backup and restore

The job history can be saved and restored using the `./bin/jenkins/backup.sh` and `./bin/jenkins/restore.sh` scripts. Executing:

    $ ./bin/jenkins/backup.sh <environment>

creates a `backup.tar.gz` file in the current directory with all the existing executions of the jobs, being `<environment>` `local` or `remote` (default).

These backup files can be restored to a running server with:

    $ ./bin/jenkins/restore.sh /path/to/backup/file <environment>

## Secrets management

We use Hashicorp's [https://www.vaultproject.io/](Vault) to manage the secrets used by the jenkins deployment. There is a script for setting up this service on a cloud environment, you can provision it with:

    $ ./bin/secrets/cloud-provision.sh <private_key_path> <openstack_credentials_path>

being `<private_key_path>` the ssh key that the slaves will use for creating images and accessing instances created from them, and `<openstack_credentials_path>` the path of the novarc file needed for accessing the cloud resources where the testbeds will be created. These secrets will be created under the vault path `secrets/jenkins/tests/ssh-key` and `secrets/jenkins/tests/openstack-credentials` respectively.

After the provisioning, a `vault-remote.txt` text file is created in the current directory containing the server keys and root token, keep it safe because they are unique for each deployment.

You can also try the deployment locally with:

    $ ./bin/secrets/local-provision.sh

`init-credentials.sh` gets the credentials from the vault server and puts them into a local or remote cluster. In order to execute this script you must be able to authenticate against the Vault server. It is executed with:

    $ ./bin/secrets/init-credentials.sh <environment>

with `<environment>` being either `local` or `remote` (default).

Finally, you can also backup and restore the credentials with the `./bin/secrets/backup.sh` and `./bin/secrets/restore.sh` scripts.


[1] https://github.com/ubuntu-core/snappy

[2] https://launchpad.net/snappy-tests-job

[3] https://wiki.jenkins-ci.org/display/JENKINS/GitHub+pull+request+builder+plugin

[4] https://github.com/ubuntu-core/snappy-cloud-image

[travis-image]: https://travis-ci.org/ubuntu-core/snappy-jenkins.svg?branch=master
[travis-url]: https://travis-ci.org/ubuntu-core/snappy-jenkins
