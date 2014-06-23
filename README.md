charm-bootstrap-wsgi
====================

This repo is a template charm for any [juju][1] deployed wsgi service.
As is, this charm deploys an example wsgi service with nagios checks and simple
rolling upgrades.

You can re-use this charm to deploy any wsgi service by updating the
playbook.yaml file. All of the wsgi functionality is provided
by a reusable wsgi-app ansible role (see roles/wsgi-app) together
with the gunicorn charm.

Disclaimer: this template does not try to explain what's possible with
either ansible or juju - but if you know a bit about both, it will
show you how you can easily use them together.


## Example - a simple application

If you've got a [bootstrapped juju environment](https://juju.ubuntu.com/docs/getting-started.html), and have [`juju-git-deploy`](https://pypi.python.org/pypi/juju-git-deploy/0.1.1) installed then:

``` bash
# Deploy example wsgi charm
$ juju git-deploy -s precise nottrobin/charm-bootstrap-wsgi wsgi-example

# Set code package URL and WSGI app location
$ juju set wsgi-example code_archive="http://stashbox.org/1522418/example-wsgi-app.v1.tar.bzip2"
$ juju set wsgi-example wsgi_application="example_wsgi:application"

# Add gunicorn server subordinate
$ juju deploy gunicorn
$ juju add-relation wsgi-example gunicorn
```

Once all the services have finished deploying (check with `juju status`),
you should be able to curl your service to see it working:

``` bash
$ juju run --service wsgi-example "curl -s http://localhost:8080"
- MachineId: "1"
  Stdout: 'It works! Revision 1'
  UnitId: charm-bootstrap-wsgi/0
```

## Upgrading the application

To upgrade the application code, simply provide a URL to the new archive (NB: the URL must be different):

``` bash
$ juju set wsgi-example code_archive="http://stashbox.org/1522420/example-wsgi-app.v2.tar.bzip2"
```

Once the app has finished installing the new archive, you should be able to see it working:

``` bash
$ juju run --service wsgi-example "curl -s http://localhost:8080"
- MachineId: "1"
  Stdout: 'It works! Revision 2'
  UnitId: charm-bootstrap-wsgi/0
```

## Nagios setup

Setting up a simple [nagios](http://www.nagios.org/) check is trivial:

``` bash
# Set expected content for the app's index page
$ juju set wsgi-example nagios_index_content="It works!"

# Add nagios charm and relation
$ juju deploy nrpe-external-master
$ juju add-relation wsgi-example nrpe-external-master

# See the output of all nagios checks
$ juju run --service wsgi-example "egrep -oh /usr.*lib.* /etc/nagios/nrpe.d/check_* | sudo -u nagios -s bash"
```

## Your custom deployment code

You can set your own archive URL or path, WSGI application location, apt dependencies
and Nagios check content using the [juju config options](https://juju.ubuntu.com/docs/charms-config.html)
in [config.yaml](config.yaml) - as partially detailed above.

If you need you can add further customisations to [playbook.yml](playbook.yml).
In addition to the wsgi-app reusable role and the optional nagios reusable role
(nrpe-external-master), it only has two tasks:

 * installing any package dependencies
 * Re-rendering the app's config file (and triggering a wsgi restart)

If you find yourself needing to do more than this, let me know :-)

For simplicity, the default example app is deployed from the charm itself with
the archived code in the charm's files directory. But the wsgi-app role also
allows you to define a code_assets_uri, which if set, will be used instead of
the charm's files directory.

The nagios check used for your app can be updated by adjusting the
check_params passed to the role in playbook.yml (or you can additionally
add further nagios checks depending on your needs).

### Note about test Dependencies

The makefile to run tests requires the following dependencies

- python-nose
- python-mock
- python-flake8

installable via:

```
$ sudo apt-get install python-nose python-mock python-flake8
```

[1]: http://juju.ubuntu.com/
[2]: http://ansibleworks.com/
