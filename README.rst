Deploy Websauna website with Ansible.

.. contents:: :local:

Introduction
============

`See more documentation here <https://websauna.org/docs/narrative/deployment/index.html>`_.

This is an Ansible playbook for automatically deploying a single server Websauna website from a git repository for Ubuntu 14.04 Linux. It allows you to do deploy your Websauna application to a fresh server, where you just received SSH credentials, within 30 minutes. Alternatively you can gracefully upgrade any existing running site.

Automatic installation sets up

* PostgreSQL

* Nginx

* uWSGI

* Celery

* Email out via upstream SMTP server and locally configured Postfix

* Supervisor startup scripts

Furthermore

* No private SSH keys are placed on a server, all SSH communication is done over SSH agent

* Websauna application files and uWSGI processes are deployment under a normal UNIX user ``wsgi``

* Firewall is set up to allow only inbound SSH, HTTP, HTTPS

* The application is deployed in the folder ``/srv/pyramid/yourapplicationpackage`` per Filesystem Hierarchy Standard

* Safely run migrations: Run database migrations first and then update codebase with compatible model files to avoid breaking an existing running site

`See documentation <https://websauna.org/docs/narrative/deployment/index.html>`_.

production.ini
--------------

The playbook does not take the ``production.ini`` from your application package, but generates one from a template (see ``websauna.site/templates/production.ini``). If you want to customize this please override the template using a local file.

production-secrets.ini
----------------------

Secrets file for the production site is not kept in version control to avoid accidental leak of confidential credentials.To copy a secrets file to the server you need to set ``local_secrets_file``. E.g.::

    local_secrets_file: ~/myapp-production-secrets.ini

If no setting is given a dummy empty ``production-secrets.ini`` is created.

Local testing
=============

With the following command a virtual server is deployed with `myapp tutorial <https://github.com/websauna/myapp>`_ from its Github repository::

    vagrant up

And to update a running VM::

    vagrant provision


Issue backlog
=============

setuptools broken:

* https://bitbucket.org/pypa/setuptools/issues/502/packaging-164-does-not-allow-whitepace

===============================
Installing Ansible and playbook
===============================

Introduction
============

:term:`Ansible` runs on your local computer and talks with the remote server over :term:`SSH`. In an ideal situation, you never need to connect to the server manually over SSH, as Ansible does all the tasks for you.

Ansible is driven by a :term:`playbook` which is effectively a linear script of commands to be run on the server. Playbooks are very human readable as is, even if you wouldn't use Ansible yourself. Playbooks are usually distributed as cloneable Git repositories.

Installation
============

Websauna's playbook ``websauna.ansible`` is provided in a separate :term:`Git` repository. See `websauna.ansible Git repository <https://github.com/websauna/websauna.ansible>`_.

Clone the repository from Github to get started with your Playbook:

.. code-block:: console

    git co git@github.com:websauna/websauna.ansible.git

Create a :term:`virtual environment` for Ansible. This must be a separate from the virtual environment of your application due to Python version differences:

.. code-block:: console

    cd websauna.ansible
    virtualenv -p python2.7 venv
    source venv/bin/activate
    pip install ansible<2.2  # Stouts.nginx is currently incompatible with latest Ansible

.. note ::

    Ansible runs on Python 2.x only. Ansible is a Red Hat product. Red Hat is committed to support Python 2.4 for their enterprise users. As long as Python 2.4 is supported, it is impossible to upgrade Ansible to support Python 3.x due to syntax incompatibilities.

.. code-block:: console

    mac:

    cd websauna.ansible
    virtualenv -p python2.7 venv
    source venv/bin/activate
    brew install openssl --force
    echo 'export PATH="/usr/local/opt/openssl/bin:$PATH"' >> ~/.zshrcenv
    LDFLAGS="-L/usr/local/opt/openssl/lib" CPPFLAGS="-I/usr/local/opt/openssl/include" CFLAGS="-I/usr/local/opt/openssl/include" pip install "ansible<2.2"

Install packaged roles we are going to use inside a cloned playbook. They will be dropped in ``galaxy`` folder inside the playbook folder:

.. code-block:: console

    ansible-galaxy install -r requirements.yml

Creating Ansible vault
======================

Create an Ansible :term:`vault` with a password. The vault is a secrets file where Ansible stores non-public configuration variables. To avoid retyping the password every time, the password is saved in plaintext in your home folder or any other safe location. The default password storing location is in ``~/websauna-ansible-vault.txt`` as configured in ``ansible.cfg``:

.. code-block:: console

    # Read a password from keyboard and store it in a file.
    # This file is configured in ansible.cfg
    read -s pass | echo $pass > ~/websauna-ansible-vault.txt

    # Create a secrets.yml vault for your project
    ansible-vault create secrets.yml

This will open your text editor and let you edit the vault in an unencrypted format.

* You do not need to add anything in this file for now. It will be filled in later in the instructions.

* Save file

* Quit your text editor to get back to the command line

Using alternative text editor with Ansible vault
------------------------------------------------

You can specify any command line compatible editor for vault editing. For example on OSX one could do:

.. code-block:: console

    # Use default OSX text edit as vault editor
    export EDITOR="/usr/bin/open -n -W -a /Applications/TextEdit.app"

    # Create a secrets.yml vault for your project using TextEdit
    ansible-vault create secrets.yml

`More information using UNIX EDITOR environment variable (Ubuntu) <http://askubuntu.com/questions/432524/how-do-i-find-and-set-my-editor-environment-variable>`_.

`More information using UNIX EDITOR environment variable (OSX) <http://stackoverflow.com/questions/3539594/change-the-default-editor-for-files-opened-in-the-terminal-e-g-set-it-to-text>`_.

====================================
Let's Encrypt certificates for HTTPS
====================================

.. contents:: :local:

Introduction
============

`Let's Encrypt <https://letsencrypt.org/>`_ is a non-profit service provising free TLS (HTTPS) certificates with automated installation process. This chapter shows how to integrate Let's Encrypt for HTTPS certificates with *websauna.ansible* playbook and Nginx.

These instructions will set up a cron job that automatically updates Lets Encrypt certificates before their 3 months expiration time is up.

Installation
============

You need ``ansible-letsencrypt`` role that is known to be compatible with Websauna playbook. In the folder where you have ``playbook.yml`` file create or append  ``requirements.yml`` with the contents:

.. code-block:: yaml

    - src: git+https://github.com/websauna/ansible-letsencrypt.git
      name: ansible-letsencrypt

Then install the requirement:

.. code-block:: console

    ansible-galaxy install -r requirements.yml

Setting up a playbook
=====================

Here are the main settings you need to change. `See fully functional playbook example <https://github.com/websauna/websauna.ansible/blob/master/playbook-letsencrypt.yml>`_.

Important variables:

.. code-block:: yaml

    - letsencrypt: on
    - ssl: on

    # Let's encrypt parameters
    - server_name: letsencrypt.websauna.org  # Your server fully qualified domain name
    - letsencrypt_webroot_path: /var/www/html
    - letsencrypt_email: mikko@opensourcehacker.net  # Your email
    - letsencrypt_cert_domains:
      - "{{ server_name }}"
    - letsencrypt_renewal_command_args: '--renew-hook "service nginx restart"'  # Ubuntu 14.04 nginx restart
    - nginx_ssl_certificate_path: "/etc/letsencrypt/live/{{ server_name }}/cert.pem"
    - nginx_ssl_certificate_path_key: "/etc/letsencrypt/live/{{ server_name }}/privkey.pem"

New role ``letsencrypt`` as:

.. code-block:: yaml

    roles:
      # ...
      - { role: Stouts.python, become: yes, become_user: root }
      - {role: ansible-letsencrypt, tags: 'letsencrypt'}
      - { role: websauna.site, tags: ['site'] }  # Core site update logic
      # ...

Rerun full playbook to make changes effective.

More information
================

`Known Good Let's Encrypt role for Ansible <https://github.com/websauna/ansible-letsencrypt>`_.

`See fully functional playbook example <https://github.com/websauna/websauna.ansible/blob/master/playbook-letsencrypt.yml>`_.
