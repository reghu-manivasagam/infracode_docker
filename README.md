Infracode in Docker Compose
CI Build Status

This docker-compose is provided as a way to allow "get up and running" quickly with infracode using Docker (based on st2-dockerfiles). It is not designed to be used in production, but rather a way to test out  Infracode and facilitate pack development.


TL;DR
docker-compose up -d
docker-compose exec st2client bash  # this gives you access to the st2 command line
Open http://localhost/ in your browser.  Infracode Username/Password by default is: st2admin/Ch@ngeMe.

Usage
Prerequisites
Docker Engine 18.09+
Docker Compose 1.12+
Compose Configuration
The image version, exposed ports, chatops, and "packs.dev" directory are configurable with environment variables.

ST2_VERSION this is the tag at the end of the docker image (ie: infracode/st2api:v3.3.0)
ST2_IMAGE_REPO The image or path to the images. Default is "infracode/". You may change this is using the Enterprise version or a private docker repository.
ST2_EXPOSE_HTTP Port to expose st2web port 80 on. Default is 127.0.0.1:80, and you may want to do 0.0.0.0:80 to expose on all interfaces.
ST2_PACKS_DEV Directory to development packs, absolute or relative to docker-compose.yml. This allows you to develop packs locally. Default is ./packs.dev. When making a number of packs, it is recommended to make a directory outside of st2-docker, with each subdirectory underneath that being an independent git repo. Example: ST2_PACKS_DEV=${HOME}/mypacks, with ${HOME}/mypacks/st2-helloworld being a git repo for the "helloworld" pack.
ST2_CHATOPS_ENABLE To enable chatops, set this variable to any non-zero value. Also ensure that your environment settings are configured for your chatops adapter (see the st2chatops service environment comments/settings for more info)
HUBOT_ADAPTER Chat service adapter to use.
HUBOT_SLACK_TOKEN If using the Slack adapter, this is your "Bot User OAuth Access Token"
Credentials
The files/htpasswd file is provided with a default username of st2admin and a default password of Ch@ngeMe. This can be changed using the htpasswd utility.

Another file (files/st2-cli.conf) contains default credentials and is mounted into the "st2client" container. If you change credentials in htpasswd, you will probably want to change them in st2-cli.conf.

Further configuration
The base st2 docker images have a built-in /etc/st2/st2.conf configuration file. Each st2 Docker image will load:

/etc/st2/st2.conf (default st2.conf)
/etc/st2/st2.docker.conf (values here will override st2.conf)
/etc/st2/st2.user.conf (values here will override st2.docker.conf)
Review st2.docker.conf for currently set values, and it is recommended to place overrides in st2.user.conf.

If you want to utilize a custom config for  Infracode Web UI (st2web container), you can do that by editing files/config.js file and mounting it as a volume inside the container as per example in docker-compose.yml.

Chatops configuration
Chatops settings are configured in the environment section for the st2chatops service in docker-compose.yml

Set ST2_CHATOPS_ENABLE to any non-zero value, then edit the various HUBOT_ variables specific to your chatops adapter. 

You will also need an st2 API key for chatops. This should be set in ST2_API_KEY.

To generate an API key,

Note: If you are standing up st2 for the first time, you may first need to start with chatops initially disabled so you can generate an API key. Once this is done, set it in ST2_API_KEY, enable chatops as per above and docker-compose restart to restart your st2 stack.

RBAC Configuration
Starting with v3.4.0 RBAC is now included, but not enabled, by default. There are some default assignments, mappings, and roles that ship with st2-docker. All the configuration files for RBAC are kept in ./files/rbac. Consult the st2 RBAC documentation for further information.

To enable RBAC you can edit st2.user.conf and add the following options:

[rbac]
enable = True
backend = default
Any changes made to RBAC assignments, mappings, or roles have to be synced in order to take effect. Normally running st2-apply-rbac-definitions will sync the files, but because all database information is not in the standard st2.conf file you need to specify the config file

To sync RBAC changes in st2client:

st2-apply-rbac-definitions --config-file /etc/st2/st2.docker.conf
LDAP is also a feature that is now included, but not enabled, by default. Roles to LDAP groups can be configured in ./files/rbac/mappings. Consult the st2 LDAP documentation for further information

Step by step first time instructions
First, optionally set and export all the environment variables you want to change. You could make an .env file with customizations.

Example:

export ST2_PACKS_DEV=$HOME/projects/infracode-packs
export ST2_EXPOSE_HTTP=0.0.0.0:80
export ST2_CHATOPS_ENABLE=1
export HUBOT_SLACK_TOKEN=xoxb-MY-SLACK-TOKEN
Secondly make any customizations to files/st2.user.conf, files/htpasswd, and files/st2-cli.conf.

Example:

To enable sharing code between actions and sensors, add these two lines to files/st2.user.conf:

[packs]
enable_common_libs = True
Third, start the docker environment:

docker-compose up -d
This will pull the required images from docker hub, and then start them.

To stop the docker environment, run:

docker-compose down
Gotchas
Startup errors
If your system has SELinux enabled you will likely see problems with st2 startup, specifically the st2makesecrets container will repeatedly restart and docker logs shows:

/bin/bash: /makesecrets.sh: Permission denied

The fix is to disable SELinux (or to put it in permissive mode).

Disable temporarily with: setenforce 0
Change to use permissive mode on the next reboot with: sed -ie 's|^SELINUX=.*|SELINUX=permissive|' /etc/selinux/config
Chatops
Chatops has been minimally tested using the Slack hubot adapter. Other adapter types may require some tweaking to the environment settings for the st2chatops service in docker-compose.yml

The git status output on the !packs get command doesn't appear to work fully.

Use docker-compose logs st2chatops to check the chatops logs if you are having problems getting chatops to work

Regular Usage


Register the pack config
If you used docker cp to copy the config in, you will need to manually load that configuration. The st2client service does not need access to the configs directory, as it will talk to st2api.

$ docker-compose exec st2client st2 run packs.load packs=git register=configs


docker-compose down --remove-orphans -v
Testing
Testing st2-docker is now powered by BATS Bash Automated Testing System. A "sidecar" like container loads the BATS libraries and binaries into a st2client-like container to run the tests

To run the tests

docker-compose -f tests/st2tests.yaml up
To do a clean teardown

docker-compose -f tests/st2tests.yaml down -v
