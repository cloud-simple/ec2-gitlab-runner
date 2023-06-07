> Instructions on how to install, configure, and upgrade **GitLab runner** as AWS EC2 instance

## General info

* Infrastructure deployment: AWS EC2 instance deployed with AWS launch template
* EC2 instance host configuration
    * OS version: Ubuntu 22.04 (on the moment of this document creation)
    * FQDN: `runner.example.org`
    * SSH access
        * Internal only — within project VPC or via VPN to the VPC
        * SSH public keys only 
            * List of SSH public keys are taken from [git@github.com:example-org/docs/team/public.git](https://github.com/example-org/docs/team/public#ssh-public-keys) repo
* **GitLab runner** details
    * Installation method: [Downloading a binary manually](https://docs.gitlab.com/runner/install/linux-manually.html#using-binary-file)
    * Software version: `15.11.0` (on the moment of this document creation)
    * Configured runner types
        * For building container images (with Docker-in-Docker)
            * Executor: [Docker](https://docs.gitlab.com/runner/executors/docker.html)
            * Pipeline tags: `share`
            * Conffigure concurrency: `15`
        * For deploying application `helm` charts (with `helm`, `kubectl`, and `aws` cli)
            * Executor: [Shell](https://docs.gitlab.com/runner/executors/shell.html)
            * Pipeline tags: `example-org`
            * Conffigure concurrency: `5`

## Installation and configuration

* To create, register, and configure **GitLab runners** follow these steps

1. Create EC2 instance using the latest version of the AWS launch template `lt-gitlab-runner-ububtu-2204` (in `us-east-2` region). On the moment of this document creation the template has the following settings
    * Launch template ID: `lt-0123456789abcdefg`
    * Launch template name: `lt-gitlab-runner-ububtu-2204`
    * Default version: `NN`
    * AMI ID: `ami-0a695f0d95cefc163` (`amazon/ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20230325`)
    * Instance type: `m5a.xlarge`
    * Security group IDs
        * `sg-abcdefgh` - default VPC security group
        *  `sg-0123456789abcdefg` - Allow access to all VPC Subnets from Management Client VPN Endpoint
    * Key pair name: `ec2init` [^a]
    * Volume 1
        * Storage type: `EBS`
        * Device name: `/dev/sda1`
        * Size (GiB): `200`
        * Volume type: `gp3`
        * IOPS: `3000`
    * Resource tags
        * Resource types: `Instances` [^b], `Volumes`, `Network interfaces`
            * Key: `Name`
            * Value: `runner.example.org`
    * Advanced details
        * Shutdown behavior: `terminate`
        * EBS optimized instance: `true`
        * Allow tags in metadata: `enabled`
        * User data: See in `user-data` dir of this repo
            * [User data content](user-data/user-data.content)
            * [User data MD5 sum](user-data/user-data.md5sum)
1. Configure all `kubectl` contexts for `gitlab-runner` linux user on the create EC2 instance:
    * Make SSH connection to the instance with `ubuntu` user and SSH key pair used for EC2 setup (`ec2init` above)
    * Switch to user `gitlab-runner` with the following command
        * `sudo -i -u gitlab-runner`
    * Setup AWS CLI credentials for the user `gitlab-runner` (`~/.aws/credentials`, and `~/.aws/config`)
    * Run the following command as the user `gitlab-runner`
        * `/home/gitlab-runner/scripts/update-kube-config.sh`
1. Register runners in **gitlab.com**:
    * Obtain registration token (`<TOKEN>`) for **GitLab runners** of `example-org` [group](https://gitlab.com/groups/example-org/-/runners)
    * Make SSH connection to the instance with `ubuntu` user and SSH key pair used for EC2 setup (`ec2init` above)
    * Switch to root with the following command
        * `sudo -i`
    * Run the following command as root
        * `/root/scripts/register-gitlab-runner.sh <TOKEN>`
1. Provide SSH access to team members for the create EC2 instance:
    * Make SSH connection to the instance with `ubuntu` user and SSH key pair used for EC2 setup (`ec2init` above) having SSH key for `git@github.com:example-org/docs/team/public.git` repo forwarded with SSH agent
    * Run the following command as user `ubuntu`
        *  `/home/ubuntu/scripts/add-users.sh`

[^a]: It corresponds to the `ubuntu` user for the Ubuntu Linux OS
[^b]: It is necessary to specify `Name` tag for the instance as host's FQDN, as it is used by user data script

* The above steps create EC2 instance, and install and configure Ubuntu Linux OS on it with all necessary packages to launch `gitlab-runner` and be able to run GitLab CI pipelines
* The following packates are installed and configured
    * `kubectl`, `helm`, `docker-ce`, `aws` cli, `gitlab-runner`
* Example configuration file for `gitlab-runner` (created by these steps on `/etc/gitlab-runner/config.toml` path) is available in `examples` dir of this repo, see [examples/config.toml](examples/config.toml)
* The foolowing cron job are also created to clean stale docker artefacts
    * `/etc/cron.hourly/docker-prune` - writes logs to `/var/log/docker-prune.log`
    * `/etc/cron.weekly/docker-rm-stales` - writes logs to `/var/log/docker-restart.log`

## Upgrade

* The **Installation and configuration** procedure above use the latest stable version of **GitLab runner** binary file
* If you need to upgrade only **GitLab runner** software version
    * Repeate the **Installation and configuration** procedure to create new EC2 instance with the latest **GitLab runner** software version
* If you need to upgrade OS version and **GitLab runner** software version
    * Change the AMI ID in the AWS launch template `lt-gitlab-runner-ububtu-2204` to correspond to new OS version
    * Repate the **Installation and configuration** procedure to create new EC2 instance with the latest **GitLab runner** software version and corresponding new OS version
* Change Route53 A-type resource records for name `runner.example.org` (in Hosted zone `example.org`) to correspond to new instance internal IP address 
* Remove the previous EC2 instance
    * Before removing the previous EC2 instance with **GitLab runners**, delete them from the registered list of **GitLab runners** for `example-org` [group](https://gitlab.com/groups/example-org/-/runners)

## Footnotes
