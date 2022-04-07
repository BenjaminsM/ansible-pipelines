# ansible-pipelines

[![CircleCI](https://circleci.com/gh/molgenis/ansible-pipelines/tree/master.svg?style=svg)](https://circleci.com/gh/molgenis/ansible-pipelines/tree/master)

## Automagic deployment of pipelines for analysis of Next Generation Sequencing data

This repo contains an Ansible playbook for deploying various pipelines for analysis of Next Generation Sequencing data:
 - [NGS_DNA](https://github.com/molgenis/NGS_DNA): pipeline developed originally used for the BBMRI Genome of the Netherlands (GoNL) project.
   The NGS_DNA pipeline was further developed @ UMCG where it is used for regular production work.
 - [NGS_Automated](https://github.com/molgenis/NGS_Automated): Automation for the above, but tailored to infra @ UMCG and RUG.

The deployment consists of the following steps:
 - Use Ansible to:
   - Deploy [Lmod - the Lua based module system](https://github.com/TACC/Lmod)
   - Deploy [EasyBuild](https://github.com/easybuilders/easybuild-easyconfigs)
   - Download and unpack reference data
   - Use EasyBuild to install software for analysis of NGS data. EasyBuild will:
     - Fetch sources
     - Unpack the sources
     - Patch sources
     - Configure
     - Compile/Build
     - Install
     - Run sanity check
     - Create module file for module system of your choice (Lmod in our case).
   - Configure the Bash environment for Lmod & EasyBuild
   - Create the basic file system layout for processing a project with various pipelines for various groups.
   - Create cronjobs for running piplines automatically for functional accounts of various groups.

## Prerequisites for pipeline deployment

 - On the managing computer Ansible 2.2.x or higher is required to run the playbook.
 - The playbook was tested on target servers running CentOS >= 6 and similar Linux distro from the ''RedHat'' family.
   When deploying on other Linux distro's the playbook may need updates for differences in package naming.
 - A basic understanding of Ansible (See: http://docs.ansible.com/ansible/latest/intro_getting_started.html#)
 - A basic understanding of EasyBuild not required to deploy the pipeline ''as is'', but will come in handy when updating/modifying the pipeline and it's dependencies.

In addition you must add to ```~/.ssh/config``` on the target host from the inventory and for the user running the playbook:
```
#
# Generic stuff: prevent timeouts 
#
Host *
	ServerAliveInterval 60
	ServerAliveCountMax 5

#
# Generic stuff: share existing connections to reduce lag when logging into the same host in a second shell
#
ControlMaster auto
ControlPath ~/.ssh/tmp/%h_%p_%r
ControlPersist 5m
```

## Prerequisites for running the pipeline

 - A basic understanding of how Linux clusters work.
   The Molgenis Compute framework used to generate scripts, 
   which can be submitted to the cluster's scheduler, 
   can support multiple schedulers.
   It comes out-of-the box with templates for Slurm and PBS/Torque.
 - Illumina data from human samples.
   - When you have data from a different sequencing platform you may need to tweak analysis steps.
   - When you have data from a different species you will need to modify the playbook to provision reference data for that species.

## Defaults and how to overrule them

The default values for variables (like the version numbers of tools to install, URLs for their sources and checksums) are stored in:
```
group_vars/all
```
When you need to override any of the defaults, then create a file with the name of a group as listed in the inventory in:
```
group_vars/[groupname]
```
You don't need to list all variables in the file in ```group_vars/```, but only the ones for which you need a different value.

IMPORTANT: Do not use ```host_vars``` as that does not work well with the dynamic inventory (see below) when working with jumphosts to reach a target server.
When a jumphost is used and its name is prefixed in front of the name of the target host, then the combined "hostname" will no longer match files or directories in ```host_vars```.
Hence you should assign the destination host to a group instead and use ```group_vars``` even when the group contains only a single host.

## Dynamic vs. static inventory

 - ```inventory.ini```: is the static inventory file.
 - ```inventory.py```: is the dynamic inventory script.

The dynamic inventory script one uses the environment variable ```AI_PROXY``` and when set prefixes the name of the specified proxy/jumphost server in front of the hostnames listed in the static inventory.
The inventory files handle only (groups of) hostnames.
Hence the inventory files do not list any other SSH connection settings / parameters like port numbers, usernames, expansion of aliases/hostnames to fully qualified domain names (FQDNs), etc.
All SSH connection settings must be stored in your ```~/.ssh/config``` file. An example ```~/.ssh/config``` could look like this:

```
########################################################################################################
#
# Hosts.
#
Host some_proxy other_proxy *some_target *another_target !*.my.domain
    HostName %h.my.domain
    User youraccount
#
# Proxy/jumphost settings.
#
Host some_proxy+*
    ProxyCommand ssh -X -q youraccount@some_proxy.my.domain -W $(echo %h | sed 's/^some_proxy[^+]*+//'):%p
Host other_proxy+*
    ProxyCommand ssh -X -q youraccount@other_proxy.my.domain -W $(echo %h | sed 's/^other_proxy[^+]*+//'):%p
########################################################################################################
```

When the environment variable ```AI_PROXY``` is set like this:
```bash
export AI_PROXY='other_proxy'
```
then the hostname ```some_target``` from inventory.ini will be prefixed with ```other_proxy``` and a '+'
resulting in:
```bash
other_proxy+some_target
```
which will match the ```Host other_proxy+*``` rule from the example ```~/.ssh/config``` file.

## Deployment: running the playbook

#### 0. Fork this repo.

Firstly, fork the repo at GitHub. Consult the GitHub docs if necessary.

#### 1. Clone forked repo to Ansible control host.

Login to the machine you want to use as Ansible control host, clone your fork and add "blessed" remote:

```bash
mkdir -p ${HOME}/git/
cd ${HOME}/git/
git clone 'https://github.com/<your-github-account>/ansible-pipelines.git'
cd ${HOME}/git/ansible-pipelines
git remote add blessed 'https://github.com/molgenis/ansible-pipelines.git'
```

#### 2. Configure Python virtual environment.

Login to the machine you want to use as Ansible control host and configure virtual Python environment in sub directory of your cloned repo:

```bash
cd ${HOME}/git/ansible-pipelines
#
# Create Python virtual environment (once)
#
python3 -m venv ansible.venv
#
# Activate virtual environment.
#
source ansible.venv/bin/activate
#
# Install Ansible and other python packages.
#
if command -v pip3 > /dev/null 2>&1; then
    PIP=pip3
elif command -v pip > /dev/null 2>&1; then
    PIP=pip
else
    echo 'FATAL: Cannot find pip3 nor pip. Make sure pip(3) is installed.'
fi
${PIP} install --upgrade pip
${PIP} install ansible
${PIP} install jmespath
${PIP} install ruamel.yaml
```

#### 3A. Run playbook on Ansible control host for Zinc-Finger or Leucine-Zipper

Only for *Zinc-Finger* and *Leucine-Zipper*:
 * Inventory: use static inventory (without jumphost).
 * Ansible control hosts:
    * For *Zinc-Finger*: use `zf-ds`
    * For *Leucine-Zipper*: use `lz-ds`

Login to the Ansible control host and:

```bash
cd ${HOME}/git/ansible-pipelines
#
# Make sure we are on the main branch and got all the latest updates.
#
git checkout master
git pull blessed master
#
# Activate virtual environment.
#
source ansible.venv/bin/
#
# Run complete playbook: general syntax
#
ansible-playbook -i inventory.ini --limit INVENTORY_GROUP playbook.yml
#
# Run complete playbook: example for Zinc-Finger
#
ansible-playbook -i inventory.ini --limit zincfinger_cluster playbook.yml
#
# Run single role playbook: examples for Zinc-Finger
#
ansible-playbook -i inventory.ini --limit zincfinger_cluster single_role_playbooks/install_easybuild.yml
ansible-playbook -i inventory.ini --limit zincfinger_cluster single_role_playbooks/fetch_extra_easyconfigs.yml
ansible-playbook -i inventory.ini --limit zincfinger_cluster single_role_playbooks/fetch_sources_and_refdata.yml
ansible-playbook -i inventory.ini --limit zincfinger_cluster single_role_playbooks/install_modules_with_easybuild.yml
ansible-playbook -i inventory.ini --limit zincfinger_cluster single_role_playbooks/create_group_subfolder_structure.yml
ansible-playbook -i inventory.ini --limit zincfinger_cluster single_role_playbooks/manage_cronjobs.yml
```

###### Changing cronjobs to fetch data on a GD cluster from another gattaca server

In the default situation:
 * Samplesheets from _Darwin_ on `dat05` share/mount:
   * sequencer/scanner must store data on `gattaca02`
   * downstream analysis on `Zinc-Finger`
   * results are stored on `prm05` share/mount
 * Samplesheets from _Darwin_ on `dat06` share/mount:
   * sequencer/scanner must store data on `gattaca01`
   * downstream analysis on `Leucine-Zipper`
   * results are stored on `prm06` share/mount

If a gattaca server and/or cluster is offline for (un)scheduled maintenance,
the lab can use the other set of infra by simply saving the rawest data on the other `gattaca` server and asking Darwin to put the samplesheet on the other `dat` share.
In this case there is no need to change any cronjob settings.

If infra breaks down and we need to link a cluster to the other gattaca server, then we need to update the corresponding cronjobs:
 * Look for the `cronjobs` variable in both `group_vars/zincfinger_cluster/vars.yml` and `group_vars/leucinezipper_cluster/vars.yml`
 * Search for cronjobs that contain `gattaca` in the command/job and depending on the situation:
   * Either change `gattaca01` into `gattaca02` or vice versa.
   * Or add a second similar cronjob for the other `gattaca` server if the cluster needs to fetch data from both `gattaca` servers.
   * Or temporarily disable a cronjob by adding `disabled: true`. E.g.:
     ```
     - name: NGS_Automated_copyRawDataToPrm_inhouse  # Unique ID required to update existing cronjob: do not modify.
       user: umcg-atd-dm
       machines: "{{ groups['helper'] | default([]) }}"
       minute: '*/10'
       job: /bin/bash -c "{{ configure_env_in_cronjob }};
            module load NGS_Automated/{{ group_module_versions['umcg-atd']['NGS_Automated'] }}-bare;
            copyRawDataToPrm.sh -g umcg-atd -s gattaca02.gcc.rug.nl"
       disabled: true
     ```
 * Run the playbook to update the cronjobs for both clusters:
   * On `zf-ds`:
   ```bash
   cd ${HOME}/git/ansible-pipelines
   git checkout master
   git pull blessed master
   source ansible.venv/bin/
   ansible-playbook -i inventory.ini --limit zincfinger_cluster single_role_playbooks/manage_cronjobs.yml
   ```
   * On `lz-ds`:
   ```bash
   cd ${HOME}/git/ansible-pipelines
   git checkout master
   git pull blessed master
   source ansible.venv/bin/
   ansible-playbook -i inventory.ini --limit leucinezipper_cluster single_role_playbooks/manage_cronjobs.yml
   ```

#### 3A. Run playbook on Ansible control host for all infra except Zinc-Finger or Leucine-Zipper

For our `gattaca` servers and all HPC clusters __*except*__ *Zinc-Finger* and *Leucine-Zipper*:
  * Inventory: use dynamic inventory with jumphost.
  * Ansible control host: use your own laptop/device

Login to the Ansible control host and:

```bash
cd ${HOME}/git/ansible-pipelines
#
# Make sure we are on the main branch and got all the latest updates.
#
git checkout master
git pull blessed master
#
# Activate virtual environment.
#
source ansible.venv/bin/
#
# Configure jumphost.
#
export AI_PROXY='name_of_jumphost'
#
# Run complete playbook: general syntax
#
ansible-playbook -i inventory.py --limit INVENTORY_GROUP playbook.yml
#
# Run complete playbook: example for Winged-Helix
#
ansible-playbook -i inventory.py --limit wingedhelix_cluster playbook.yml
#
# Run single role playbook: examples for Winged-Helix
#
ansible-playbook -i inventory.py --limit wingedhelix_cluster single_role_playbooks/install_easybuild.yml
ansible-playbook -i inventory.py --limit wingedhelix_cluster single_role_playbooks/fetch_extra_easyconfigs.yml
ansible-playbook -i inventory.py --limit wingedhelix_cluster single_role_playbooks/fetch_sources_and_refdata.yml
ansible-playbook -i inventory.py --limit wingedhelix_cluster single_role_playbooks/install_modules_with_easybuild.yml
ansible-playbook -i inventory.py --limit wingedhelix_cluster single_role_playbooks/create_group_subfolder_structure.yml
ansible-playbook -i inventory.py --limit wingedhelix_cluster single_role_playbooks/manage_cronjobs.yml
```
