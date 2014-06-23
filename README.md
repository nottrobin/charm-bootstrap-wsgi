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

If you've got a [bootstrapped juju environment](https://juju.ubuntu.com/docs/getting-started.html), and have [`juju-git-deploy`](https://pypi.python.org/pypi/juju-git-deploy/0.1.1) installed then you can deploy an example WSGI application as follows:

``` bash
# Deploy example wsgi charm
juju git-deploy -s trusty nottrobin/charm-bootstrap-wsgi wsgi-example

# Set code package URL and WSGI app location
juju set wsgi-example code_archive="http://stashbox.org/1522418/example-wsgi-app.v1.tar.bzip2"
juju set wsgi-example wsgi_application="example_wsgi:application"

# Add gunicorn server subordinate
juju deploy cs:~webteam-backend/trusty/gunicorn
juju add-relation wsgi-example gunicorn
```

Once all the services have finished deploying (check with `juju status`),
you should be able to curl your service to see it working:

``` bash
juju run --service wsgi-example "curl -s http://localhost:8080"

# Expected output
It works! Revision 1
```

## Upgrading the application

To upgrade the application code, simply provide a URL to the new archive (NB: the URL must be different):

``` bash
juju set wsgi-example code_archive="http://stashbox.org/1522420/example-wsgi-app.v2.tar.bzip2"
```

Once the app has finished installing the new archive, you should be able to see it working:

``` bash
juju run --service wsgi-example "curl -s http://localhost:8080"

# Expected output
It works! Revision 2
```

## Nagios setup

Setting up a simple [nagios](http://www.nagios.org/) check is trivial:

``` bash
# Set expected content for the app's index page
juju set wsgi-example nagios_index_content="It works!"

# Add nagios charm and relation
juju deploy cs:~webteam-backend/trusty/nrpe-external-master
juju add-relation wsgi-example nrpe-external-master
```

Once the service is ready, you can see the output of all nagios checks as follows:

``` bash
juju run --service wsgi-example "egrep -oh /usr.*lib.* /etc/nagios/nrpe.d/check_* | sudo -u nagios -s bash"

# Expected output
DISK CRITICAL - free space: / 12 GB (12% inode=92%);| /=90GB;81;87;0;109
HTTP OK: Status line output matched " 200 OK" - 144 bytes in 0.002 second response time |time=0.001863s;;;0.000000 size=144B;;;0
OK - load average: 1.13, 1.14, 0.96|load1=1.130;8.000;15.000;0; load5=1.140;8.000;15.000;0; load15=0.960;8.000;15.000;0; 
SWAP OK - 100% free (0 MB out of 0 MB) |swap=0MB;0;0;0;0
PROCS OK: 39 processes | procs=39;150;200;0;
USERS OK - 0 users currently logged in |users=0;20;25;0
PROCS OK: 0 processes with STATE = Z | procs=0;3;6;0;
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
