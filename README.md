# Template repository for AWS scripts

This repository contains scripts to help you launch and configure AWS instances.  This repository includes three core template scripts that you can configure for your needs:

- `start`: Launch a new AWS instance from a specified launch template.  This script lets you launch instances from the command line and ensures that those instances are configured consistently.  For example, the script will attach a specified persistent EBS volume to your new instance.
- `setup`: Setup a new AWS instance.  This script will perform routine setup steps on a newly launched AWS instance, including the following:
  - Create a new user with your username.
  - Install a standard set of packages.
  - Copy standard configuration files.
  - Mount an attached EBS volume.
- `keys`: Start an SSH agent and add a list of keys to it.  When you use SSH-agent forwarding to connect to a machine, these keys will be available to use.

These scripts are designed to work on Linux or macOS.

Once these scripts are configured, you can use them as follows to launch and setup a new AWS instance:

```console
./start  # At the end, prints the hostname of the newly launched instance
./setup <instance_hostname>
```

To make your SSH keys available from your instance, ensure you have run the `./keys` script, and then use SSH-agent forwarding when you connect to your instance with SSH.
> [!NOTE]
> You can enable SSH-agent forwarding either using the `-A` command-line flag with `ssh` or by setting `ForwardAgent yes` in the appropriate entry in your SSH config.

The following sections describe how to configure and use these scripts for your use case.  These instructions assume you already have an AWS account.

## `start`: Launching a new instance

### Prerequisites

To use the `start` script, you will first need to do the following steps:

- Install and setup AWS CLI version 2 on your local machine: <https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html>.
- Create a launch template for launching new instances: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-launch-template.html>.
- *Recommended, but optional:* Create a persistent, empty EBS volume to store data across spot instances: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-creating-volume.html>.

### Configuring the `start` script

The `start` script uses a ***launch configuration** to specify how to launch instances.  By default, `start` uses the launch configuration `example-fleet-config.json`, which launches a single spot instance from a specified launch template that is terminated when interrupted.

You can configure the `start` script with this default configuration file as follows:

- ***Mandatory:*** In `example-fleet-config.json`, replace `<Your Launch Template ID>` with the ID of your launch template.
- In `example-fleet-config.json` at `LaunchTemplateConfigs[0].Overrides[0].InstanceType`, replace `c5.4xlarge` with your preferred instance type.
- ***To automatically attach a separate EBS volume:***
  - In `example-fleet-config.json`, make sure that `LaunchTemplateConfigs[0].Overrides[0].SubnetId` matches the subnet ID of the Availability Zone that contains your EBS volume.
  - In the `start` script itself, set the value of `PERSISTENT_VOLUME_ID` to match the ID of your EBS volume.

> [!NOTE]
> If you wish for the `start` script to choose between multiple possible instance types, create multiple entries within `LaunchTemplateConfigs[0].Overrides`, one for each instance type the script may use.  For more documentation and examples of launch configurations, see <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-examples.html>.

### Running the script

Once configured, simply run the `start` script to launch your AWS instance:

```console
./start
```

The script will report its progress in launching the instance as it goes.  Once the instance is launched, the script will print the hostname of the newly launched instance.

> ![NOTE]
> You can create different configuration `json` files to launch instances and specify which configuration file `start` should use via the `--config` command-line flag.

For more information on using the `start` script, run:

```console
./start -h
```

## `setup`: Setup a new AWS instance

### Configuring the `setup` script

Configure the `setup` script as follows:

- ***Mandatory:*** Set the `AWS_KEY_PAIR` variable to match the filename on your system of the private key for your AWS key pair, such as `my_aws_key.pem`.  The `setup` script assumes that this file is located inside `~/.ssh/`.
- ***Mandatory:*** Set the `AWS_USER_PRIVATE_KEY` variable to match the filename of the private key for logging to your AWS instance in as your preferred username, e.g., `id_ed25519`.  The `setup` script assumes that this file and the corresponding public key — such as `id_ed25519.pub` — are both located inside `~/.ssh/`.
- Set the `AWS_USER` variable to be the username you wish to use on your AWS instance.  If left unmodified, the `setup` script will set this variable to `${USER}`, that is, your username on your local machine.
- Populate the `PACKAGES` variable with a space-separated list of packages you wish to install by default on any new AWS instance.
- ***To automatically mount a separate EBS volume:*** Set the `PERSISTENT_VOLUME_MOUNT` variable to the path where your separate EBS volume should be mounted.
- Populate `home_config` with any files --- such as `.gitconfig`, `.vim/`, or `.emacs` --- that you wish to automatically copy into `AWS_USER`'s home directory on the instance.
  - To automatically append any content onto an existing hidden file — such as `.bashrc` — within `AWS_USER`'s home directory, place the text to append inside a not-hidden ***addon*** file within `home_config` with the corresponding name — such as `bashrc-addon`.

> [!WARNING]
> Because of potential security risks, I recommend that you ***do not*** create a `.ssh/` directory inside `home_config` that contains SSH keys you wish to use from the instance.  To make SSH keys available when you're using the instance, use SSH-agent forwarding and the `keys` script.

#### Additional configuration

The `setup` script assumes the AWS instance is running a recent Ubuntu AMI.  For other AMI's, you may need to modify the variables `AWS_ROOT_USER`, `PACKAGE_MANAGER_UPDATE_CMD`, and `PACKAGE_MANAGER_INSTALL_CMD`; the `adduser` and `usermod` commands for setting up a new user on the instance; and the commands for quiescing the instance.

### Running the `setup` script

Once configured, simply run the `setup` script with your AWS instance hostname to setup a newly launched AWS instance:

```console
./setup <instance_hostname>
```

where, `<instance_hostname>` is the hostname printed when the `start` script terminates.

By default, `setup` will perform all setup steps, creating the user `$USER` on the instance.  You can specify command-line flags to change the username that `setup` will create or restrict `setup` to perform specific setup steps only.  For more information on using the `setup` script, run:

```console
./setup -h
```

## `keys`: Add SSH keys to an SSH agent

### Configuring the `keys` script

Populate the `KEYS` variable in the `keys` script with a space-separated list of SSH-key names that you would like to make available on any instance you SSH to.

### Running `keys`

Once configured, simply run the `keys` script to start your SSH agent and add your keys to it:

```console
./keys
```

Once you have run `./keys` once, your SSH agent will be setup for any instances you SSH into until you next logout of your local machine.
