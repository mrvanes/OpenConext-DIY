# OpenConext-DIY
Creating an IdP to attach to a SAML consuming authentication service is a simple, yet
common task that can be easily automated.

This project has the following deployment goals:
- installation of SimpleSAMLphp
- configuration of a set of well-known logins for testing purposes
- configuration of LetsEncrypt for deployments with public internet access


SimpleSAMLphp
=============
From the [SimpleSAMLphp website](https://simplesamlphp.org/):

    SimpleSAMLphp is an award-winning application written in native PHP that deals with authentication.
    The project is led by UNINETT, has a large user base, a helpful user community and a large set of
    external contributors. The main focus of SimpleSAMLphp is providing support for SAML 2.0 as a
    Service Provider (SP) and/or SAML 2.0 as an Identity Provider (IdP)

The final product of this project is a complete SimpleSAMLphp IdP installation that uses the
'example-auth' module to allow login using a set of well-known logins as defined on:

    https://diy-idp.pilots.aarc-project.eu/simplesaml/showusers.php

These users combine a diverse set of unicode characters and entity definitions usable in testing the
proper implementations of SAML consuming service providers. The configuration of these users is
present inside the SimpleSAMLphp installation in a JSON encoded file called: `simplesamlphp/config/logins.json`,
which is copied from the ansible deployment to its final destination upon provisioning.
The original file can be found in the group_vars directory:

    group_vars/logins.json

You can add additional users and attributes to this file and (re)provision the IdP.

The final product is a single-server installation and not useable in production environments.

Setup
=====
The provisioning script was developed for deployment to an Debian Bookworm (12)
installation.

All other required packages are installed automatically.

The installation will use the default PHP version (8.2). You can make this work for other versions
as well (PHP5.6 for example) by adjusting the package requirements in the `common` role. Some tweaking
may apply in the `apache` role as well to get the right modules enabled.

Configuration
=============
The basic configuration options are available in the `group_vars/idp.yml` file. More options are
available for each of the `roles`, see the default variable definitions of these roles.

Basic options are:

*Basic Site Information*
These define the basic FQDN hostname of the target installation and the SAML consuming service linked
by default to this IdP.

    # Site information
    idp_hostname: idp.example.org
    idp_sp: https://sp.example.org


*SimpleSaml*
These configuration values determine the basic SimpleSAMLphp installation. The SimpleSAMLphp salt is created
dynamically and will change on redeployment.

    ssp_version: 1.15.4
    ssp_hostname: "{{ idp_hostname }}"
    ssp_subject: "{{ cert_subject }}"
    ssp_days_valid: "{{ cert_days_valid }}"
    ssp_key: "{{ idp_hostname }}.pem"
    ssp_certificate: "{{ idp_hostname }}.crt"
    ssp_auth_admin_password: "changethispassword"
    ssp_technicalcontact_name: Contact
    ssp_technicalcontact_email: postmaster@{{ idp_hostname }}
    ssp_managingcontact_name: Manager
    ssp_managingcontact_email: webmaster@{{ idp_hostname }}
    ssp_sp_metadata: "{{ idp_sp }}/authentication/sp/metadata"
    ssp_sp_consumer: "{{ idp_sp }}/authentication/sp/consume-assertion"
    ssp_title_suffix: "OpenConext-DIY"

The 'ssp_title_suffix' option allows differentiating between various default SimpleSaml installations. Comment
this option out to disable generating the header suffix.

*Certificates*
By default, the hostname for the certificate is the same as that of the installation. The certificate subject
is only the hostname and does not contain specific information. This only concerns the self-signed certificate
for a deployment to a private host, so has no real world implications.
    cert_hostname: "{{ idp_hostname }}"
    cert_subject: "/CN={{ cert_hostname }}"


Default configuration variables for the `letsencrypt` role. This role is based on the LetsEncrypt ansible
playbook by Jason Robinson: [ansible-letsencrypt](https://github.com/jaywink/ansible-letsencrypt),
Copyright (c) 2017 Jason Robinson.

    letsencrypt_email: idp@example.org
    letsencrypt_domain: "{{ idp_hostname }}"
    letsencrypt_request_www: false

A self-signed certificate for SimpleSAMLphp is created automatically, using the same variables by default as
defined for the self-signed certificate of the HTTPS/SSL configuration.
The following configuration values determine the basic installation directory of the SimpleSAMLphp website on
the host target. These are Apache site configuration settings.

    ssl_hostname: "{{ idp_hostname }}"
    ssl_webmaster: "webmaster@{{ ssl_hostname }}"
    ssl_docroot: "{{ ssp_dir }}/www"


Provisioning
============
After adjusting all relevant configuration values in `group_vars/idp.yml`, the Ansible inventory file needs to
be populated. The inventory file defines a group `idp` of which all target machines are a member. The target
machine IP address is defined at the top of that inventory file.

Then provision the application by running:

    ansible-playbook -i inventory openconext-diy.yml

During provisioning, the roles and tasks will:

- try to find out if the target machine has a publicly accessible network address
- if so, a LetsEncrypt certificate is requested automatically and configured
- if not, a self-signed certificate is configured for SSL
- SimpleSAMLphp salt and certificate are generated on the fly
- HTTP traffic is redirected to HTTPS for SimpleSAMLphp
- only the SimpleSAMLphp site is enabled, the default sites (default and 000-default) are disabled

After provisioning, the metadata is available at:

    https://{{ idp_hostname }}/module.php/saml/idp/metadata

You can use this link to configure service providers to accept this IdP.

Ansible version 2.5 or later is required.

Vagrant
=======
A `Vagrantfile` is provided for easy vagrant provisioning. Please adjust any overrides for the default
`group_var/idp.yml` configuration in that file. Due to a problem with the network interface
names in Ubuntu 15.10, 16.04, 16.10 and (presumably) later versions, the `vagrant up` command will fail
on network enabling. However, the box is provisioned (as far as vagrant is concerned) correctly
nonetheless. Run the following commands:

    vagrant up
    vagrant provision

to get the vagrant machine up and running. The `VagrantFile` uses the VirtualBox provider by default.

Docker
======
A basic `Dockerfile` is available to install this IdP on a Docker container. Due to Docker networking
configuration and setup, this installation knows neither hostname nor IP address, so additional
configuration after provisioning is required.

The `Dockerfile` mounts the SimpleSAML metadata directory. In the `docker-run` script file, the Docker
image is build and the run command mounts the local `docker/metadata` directory to the container,
allowing local edits to appear in the container metadata. You can use this as a starting point for
configuring and running your own containers.

Disclaimer
==========
The resulting installation is provided as-is and should not be used for anything other than test purposes.
The SimpleSAMLphp installation, Apache configuration and system firewall were not hardened for production
use in any way and using the result for real life production circumstances is strongly discouraged.
