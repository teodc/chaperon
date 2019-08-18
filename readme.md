## Chaperone

> ⚠️ Work in progress

Manage your Laravel app nodes with Ansible.

## Requirements

### On the control machine:

- SSH
- Git
- Ansible (\^2.4)
- coreutils (on macOS only)

### On the managed nodes:

- Ubuntu 18.04
- SSH
- Python (\^3.0)

## Setup

> By default, Chaperone allows you to manage two environments for your nodes: `staging` and `production`

### App environment file

Add a copy of your Laravel app's `.env` file in the `files` folder for each environment:

```
cp path/to/my/app/.env files/staging/.env
cp path/to/my/app/.env files/production/.env
```

Edit each copied `.env` file to match its environment configuration.


### Hosts Inventory

Describe your hosts inventory by copying & editing the `.dist` inventory files in the `hosts` folder:

```
cp hosts/staging.yml.dist hosts/staging.yml
cp hosts/production.yml.dist hosts/production.yml
```

### SSH key pair

Create a new SSH key pair for each environment in the `files` folder:

```
ssh-keygen -t rsa -b 4096 -C "staging@acme.com"
ssh-keygen -t rsa -b 4096 -C "production@acme.com"
```

Keep the default name `id_rsa` and generate it directly in `files/<environment>` of the Chaperone's folder.

You should end up with two keys per environment, a public one and a private one:

- `~/workspace/chaperone/files/<environment>/id_rsa`
- `~/workspace/chaperone/files/<environment>/id_rsa.pub`

Then, go to your GitHub app repository and add the generated key as a deploy key.
This way, your nodes will be able to checkout your project once they've been provisioned.

## Usage

### Using the chaperone bin

A bash executable located in the `/bin` is here to help you chaperone your nodes:

```
bin/chaperone help
```

### Using the ansible-playbook command

4 playbooks are provided for 4 different operations:
- `provision.yml`: Configure your nodes.
- `deploy.yml`: Deploy the app on your nodes.
- `pull.yml`: Pull the latest version of the app on your nodes.
- `migrate.yml`: Run the outstanding database migrations (run only on one node).
- `maintenance/down.yml`: Put the app into maintenance mode.
- `maintenance/up.yml`: Bring the app out of maintenance mode.
- `maintenance/clean-files.yml`: Clean the application temporary files & logs on each node.

Of course they should be played in this respective order.

To run the playbooks on the managed nodes, use the `ansible-playbook` command:

```
ansible-playbook -i hosts/{environment}.yml playbooks/{playbook}.yml
```

Set the `{environment}` and the `{playbook}` according to the environment you want to target and the operation you want to run.

To allow the controller machine to connect to the managed nodes via SSH, you might need to specify the private key to use with the `--private-key` option.

```
ansible-playbook -i hosts/{environment}.yml playbooks/{playbook}.yml --private-key ~/.ssh/pem/private-key.pem
```

Some roles will copy files (configurations, keys, etc.) stored in the script sources to the node. These files will most likely be encrypted for security reasons.

In order for the controller to decrypt them before copying them, you need to use the `--vault-id @prompt` option. You will then be asked to provide the encryption password before running the role:

```
ansible-playbook -i hosts/{environment}.yml playbooks/{playbook}.yml --vault-id @prompt
```

Use the `--limit` option if you want to run a playbook on a single node or a group of nodes:

```
ansible-playbook -i hosts/{environment}.yml playbooks/{playbook}.yml --limit {node}
```

Use the `--check` option to run a playbook in "dry mode" and debug its execution:

> Combine it with `-vvv` to get a more detailed output

```
ansible-playbook -i hosts/{environment}.yml playbooks/{playbook}.yml --check -vvv
```

The `--forks` option can help decrease the playbook execution time if you run it against many hosts (default is `5`):

```
ansible-playbook -i hosts/{environment}.yml playbooks/{playbook}.yml --forks 20
```

For more details on it's usage, check the `ansible-playbook` command's [documentation](http://docs.ansible.com/ansible/latest/ansible-playbook.html).

> Notes:
> - Environment specific files must share the same structure between different environments. Only values may differ.
> - Roles are common to all environments
> - Group variables are common to all environments. You can use the host file to set environment and group specific vars.
> - Sensitive files and variables are encrypted via Ansible Vault.
> - The `migrate` playbook should be run only on one node.

### Example:

```
ansible-playbook -i hosts/staging.yml playbooks/provision.yml --private-key ~/.ssh/pem/staging.pem --vault-id @prompt
```

## Troubleshooting

If the script fails:
- Check that the managed node IPs defined in the inventory file are correct.
- Check that the managed nodes can be accessed via SSH by the controller machine.
- Verify that the required packages are installed on the control machine & on the managed nodes.
