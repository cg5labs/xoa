## Local Vagrant workflow

The repository root is the only allowed working directory for Vagrant commands.

Allowed commands:

- `vagrant up`
- `vagrant destroy --force`

Run them only from the repository root. Do not ask for confirmation before destroying the VM.

## Install process with Ansible

* Install package dependencies
* Render config-files from Jinja templates
* Invoke Xen Orchestra Appliance (XOA) installer xo_install.sh to build the application with Yarn.  
  Yarn build log is captured at /var/log/yarn-build.log
* XOA configured to run on tcp/8000
* Nginx proxy installs and configures a reverse proxy to XOA on tcp/443