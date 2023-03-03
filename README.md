# MDR-X Deployment
This project is concerned with the deployment of the MDR-X system to the NIMS Google Cloud infrastructure.

(The codebase for the MDR-X web application is managed in a [separate GitHub repository)](https://github.com/antleaf/mdr-x-development).

## Project management

The development tasks for this project are managed in a project board (kanban) in this repository - *[Google Cloud Deployment](https://github.com/antleaf/mdr-x-deployment/projects/1)*.


## Design overview

Almost everything is deployed using Ansible playbooks, with various components split out into separate Ansible roles. The playbooks support multiple deployment environments, each corresponding to a Google Cloud Platform project.
Environment-specific configuration is found in [vars/environment/](https://github.com/antleaf/mdr-x-deployment/tree/main/vars/environment), and defines the GCP project details, the top-level domain name, and the instances in that environment.

## Architecture

This is described in more detail in the [Architecture document](https://github.com/antleaf/mdr-x-deployment/blob/main/docs/architecture.md).


## Usage

### Setting up Ansible

Install the required collections from ansible-galaxy:

```shell
$ ansible-galaxy -r requirements.yml
```

### Authentication

To authenticate with the Google Cloud API, you should ensure that the
`GOOGLE_APPLICATION_CREDENTIALS` environment variable is unset, and then run:

```shell
$ gcloud auth login  # Authenticates the `gcloud` tool to GCP
$ gcloud auth application-default login  # Sets up the Application Default
                                         # Credentials for Ansible to talk to GCP
```


### Configuring a Google Cloud project

You will need access to the pre-existing Google Cloud Platform project into
which you intend to deploy. Your user will need the following
[IAM roles](https://console.cloud.google.com/iam-admin/iam):

* Editor
* Kubernetes Engine Admin


### Google Cloud Build configuration

Enable the [Cloud Build API](https://console.cloud.google.com/marketplace/product/google/cloudbuild.googleapis.com) for
your project.

You will need to ensure that the MDR-X application is set up in
[Cloud Build](https://console.cloud.google.com/cloud-build/dashboard). To do this, set up a trigger with the following
details:

* Name: mdrx
* Event: push to branch
* Source repository: antleaf/mdr-x-development (GitHub App)
* Branch: `.*`
* Configuration type: Autodetected
* Configuration location: Repository

You should trigger a manual build, which will will create an image artifact and upload it to your project's Google
Container Repository, where it can be accessed from the Kubernetes cluster.


### DNS configuration

Each environment defined in `vars/environment/` has a `zone` key containing two variables:

* `dns_name`
* `google_site_verification`

The first of these is the root domain name where the deployment will appear on the web.

The second of these is the Google Search Console verification DNS TXT record. To use domain-name-based Google Cloud
Storage buckets, the deploying user must be set up as an owner of the `dns_name` domain in the [Google Search
Console](https://search.google.com/search-console). Within the Search Console, you need to select "Add property", and
add a domain in the dialog that appears. A "Verify domain ownership via DNS record" dialog will appear, containing a
string that starts `google-site-verification=`. You should copy this into the `google_site_verification` variable.

When both of these are done, run the `dns` playbook tag:

```shell
$ ansible-playbook mdrx.yml --limit gcp-test --tags dns
```

Finally, you will need to delegate DNS from the parent nameserver to the Google nameservers listed in the `NS` section
in [Cloud DNS](https://console.cloud.google.com/net-services/dns/zones) once you have run the playbook's `dns` tag.

Return to the Search Console and ensure the ownership has been verified. You may need to wait for your DNS changes to
propagate.

DNS needs to be configured before further deployment so that SSL certificates can be obtained from LetsEncrypt and
domain-name-based buckets can be created.


### Outgoing email configuration

GitLab needs to be able to send tugoing email over SMTP for account management purposes. To configure email, edit the
following keys in your `vars/environment/<name>.yml` file:

* `smtp_host`: hostname of the SMTP server
* `smtp_port`: port to connect to (default `465`)
* `smtp_username`: username to authenticate as to the SMTP server

Other settings variables can be found in `tasks/gitlab/main.yml`.

In the `vaults/environment/<name>.yml`, you set the SMTP password:

* `smtp_password`


### Deployment

Once you're ready, you can deploy to an environment with e.g.:

```shell
$ ansible-playbook mdrx.yml --limit gcp-test
```

The `gcp-test` corresponds to a host in the `hosts` file, which in turn
references an environment.


### Finding the root GitLab password

Each GitLab instance will have its own initial password for the `root` user.
You can discover the password using the `display-gitlab-root-password.yml`
playbook:

```shell
$ ansible-playbook display-gitlab-root-password.yml --limit gcp-test
Which instance name?: alpha

[...]

TASK [Display root password] *****************************************************************************************************************************************************************************************************************
ok: [gcp-test] => {
    "msg": "The initial GitLab root password for the alpha instance is: [...]"
}
```
