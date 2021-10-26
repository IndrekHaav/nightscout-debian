This directory contains an [Ansible](https://www.ansible.com/) role and example playbook to set up Nightscout on a Debian-based server.

The role has been tested against Debian 11 "Bullseye" and Ubuntu Server 20.04. Compatibility with other versions or distros is not guaranteed.

## How to use

If you already have an existing Ansible setup, feel free to grab just `roles/nightscout` and use it in your playbooks.

Otherwise, open `inventory/hosts.yml` and enter your Nightscout server's IP address, your username on the server and the filename of your SSH key.

Then, open `playbooks/nightscout.yml` and set some Nightscout configuration parameters. Consult `roles/nightscout/defaults/main.yml` for a list of variables that can be set.

> **Note:** Currently, the role installs Nightscout with only the bare minimum configuration necessary to get the web UI up and running. This might change in the future, but for now you can use the configuration file at `/opt/nightscout/cgm-remote-monitor/.env` on your Nightscout server to perform further configuration (repeated runs of the playbook will not overwrite this file). Consult the [official docs](https://github.com/nightscout/cgm-remote-monitor#environment) for a full list of configuration parameters.

Next, install Ansible requirements with the following command:

```shell
$ ansible-galaxy install -r requirements.yml
```

Finally, run the playbook with the following command:

```shell
$ ansible-playbook playbooks/nightscout.yml
```

The role will automatically generate and use random passwords for MongoDB (both the admin account and Nightscout user) as well as Nightscout's API key. The passwords will be stored in the `secrets/` directory (which is created during the playbook run), so make sure to copy them to someplace safe. Ideally the files should be present on repeated runs of the playbook, but nothing bad will happen if they're not.

To update Nightscout, it is not necessary to run the full playbook again. The following command will run just the tasks that download and install a fresh copy of the Nightscout source code:

```shell
$ ansible-playbook playbooks/nightscout.yml -t update
```
