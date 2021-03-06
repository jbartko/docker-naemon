# Docker naemon image

A turn-key naemon image ready for most deployment situations. The key features are:

1. Externalized naemon configuration through the `/data` volume
2. Configurable Thruk SSL support
3. LDAP authnentication support for Thruk
4. External SMTP server support via [ssmtp](https://wiki.archlinux.org/index.php/SSMTP)
5. Jabber notification support via [sendxmpp](http://sendxmpp.hostname.sk) (See below)

Each of these features can be enabled independently, so you can pick and choose the features you'd like to use.

__Warning: Since naemon updates can include significant one-way changes, you shouldn't use the "latest" tag outside of testing. All examples in this documentation expect that the you will replace "TAG" with a recent version__

## Quick start

The command below will setup a naemon container, with no SSL or LDAP support

```
 docker run --name naemon -h naemon -d -p 80:80\
  -e SMTP_HOST="gmail.smtp.com" -e SMTP_PORT=587 -e SMTP_USER=user@gmail.com\
  -e SMTP_PASS=pass -e NOTIFICATION_FROM="naemon@example.com"\
  -v /somepath/naemon_mnt:/data xetusoss/naemon:TAG
```

## Examples for different scenarios

#### (1) HTTPS support

Create a public/private key pair and place them in data mount under `crt.pem` and `key.pem`. If this key pair was signed by a non-standard CA, include the CA certificate as `ca.pem`. Remove the `WEB_SSL_CA` variable if not using an internal CA.

```
 docker run --name naemon -h naemon -d -p 443:443\
  -e SMTP_HOST="gmail.smtp.com" -e SMTP_PORT=587 -e SMTP_USER=user@gmail.com\
  -e SMTP_PASS=pass -e WEB_SSL_ENABLED=true -e WEB_SSL_CA=/data/ca.pem\
  -e NOTIFICATION_FROM="naemon@example.com"\
  -v /somepath/naemon_mnt:/data xetusoss/naemon:TAG
```

#### (2) LDAP support with group filter

The command below will setup a container with Thruk configured to authenticate users against an LDAP backend, but restricts access using the group filter.


```
 docker run --name naemon -h naemon -d -p 80:80 \
  -e SMTP_HOST="gmail.smtp.com" -e SMTP_PORT=587 \
  -e SMTP_USER=user@gmail.com -e SMTP_PASS=pass \
  -e WEB_LDAP_HOST=ldap.example.com\
  -e 'WEB_LDAP_BIND_DN="uid=naemonuser,dc=example,dc=com"'\
  -e WEB_LDAP_BIND_PASS=pass -e WEB_LDAP_BASE_DN="dc=example,dc=com"\
  -e WEB_USERS_FULL_ACCESS=true\
  -e 'WEB_LDAP_FILTER="(memberof=cn=naemonusers,cn=groups,dc=example,dc=com)"'\
  -e NOTIFICATION_FROM="naemon@example.com"\
  -v /somepath/naemon_mnt:/data xetusoss/naemon:TAG
```

#### (3) Jabber support with SSL

The command below will setup a container configured to speak to a Jabber server over SSL for notifications, and has no support for email notifications.

```
 docker run --name naemon -h naemon -d -p 80:80 \
  -e JABBER_USER=myuser@im.corp.com -e 'JABBER_PASS=secret' -e JABBER_PORT=5223
  -v /somepath/naemon_mnt:/data xetusoss/naemon:TAG
```

## Jabber support

Jabber notification commands are added to the naemon configuratoin in the `notify_jabber.cfg`. By default, there are two sets of commands which can be used for different scenarios, and their names are fairly intuitive. 

1. notify-[host|service]-by-jabber-chatroom-ssl
2. notify-[host|service]-by-jabber-user-ssl
3. notify-[host|service]-by-jabber-chatroom
4. notify-[host|service]-by-jabber-user

Of course, these commands can't work without some additional configuration. Below is an example of a chatroom contact that should receive messages to an jabber over an SSL connection. You would need to set this up manually after starting the the container:

```
define contact {
  contact_name                    sysadmins
  alias                           System Admins
  use                             generic-contact
  email                           systemssupport@corp.com
  address1	                      naemon-notifications@conference.im.corp.com
  host_notification_commands      notify-host-by-jabber-chatroom-ssl
  service_notification_commands   notify-service-by-jabber-chatroom-ssl
}

```
## Available Configuration Parameters

There are two categories of configuration parameters - __container__ and __initial setup__. The container parameters represent the all the configuration that is not stored in files on the `/data` volume. These parameters will be re-evaluated and applied on every container replacement, though this is transparent to the end user.

Initial setup parameters, on the other hand, are only applied when a new (empty) `data` volume is used. If the container is replaced (say with a newer version) and the data volume is not replaced, then the initial setup parameters are ignored.

### Container parameters

#### SMTP support
* __SMTP_HOST__: The SMTP host to send notifications through. No default value.
* __SMTP_PORT__: The port to use on the SMTP host. Default is `25`.
* __SMTP_LOGIN__: The username to use for SMTP authentication, only needed if the SMTP server requires authentication. No default.
* __SMTP_PASS__: The password for SMTP authentication, only needed if the SMTP server requires authentication. No default.
* __SMTP_USE_TLS__: Use TLS for SMTP connections. Default is `true` if the SMTP_PORT is anything other than 25, otherwise `false`.

#### HTTPS support
* __WEB_SSL_ENABLED__: Use HTTPS for the Thruk web ui. Default is `true` if `WEB_SSL_CERT` and `WEB_SSL_KEY` are defined, otherwise `false`.
* __WEB_SSL_CERT__: The certificate path for the SSL certificate to use with Thruk, must be in the PEM format. Default is `/data/crt.pem`.
* __WEB_SSL_KEY__: The key path for the SSL certificate to use with Thruk, must be in the PEM format and have no password. Default is `/data/key.pem`.
* __WEB_SSL_CA__: The CA cert path to use with Thruk, must be in the PEM format. No default value.

#### LDAP support
* __WEB_LDAP_AUTH_ENABLED__: Enable LDAP authentication for the Thuk UI. Default is `true` if `WEB_LDAP_HOST` is defined, otherwise `false`.
* __WEB_LDAP_HOST__: The LDAP host to authenticate against. No default value.
* __WEB_LDAP_SSL__: Enable SSL with the LDAP module. Default is `false`.
* __WEB_LDAP_SSL_VERIFY__: Enable certificate verification for the SSL certificate used by the LDAP server. Default is `true` if `WEB_LDAP_SSL_CA` is defined, otherwise `false`
* __WEB_LDAP_PORT__: The port to communicate with the LDAP host on. Default is `389` if `WEB_LDAP_SSL` is `false`, and `686` if `true`.
* __WEB_LDAP_BIND_DN__: The bind dn to use for LDAP authentication. No default value.
* __WEB_LDAP_BIND_PASS__: The password to use with the bind dn for LDAP authentication. No default value.
* __WEB_LDAP_BASE_DN__: The base dn for LDAP authentication. No default value.
* __WEB_LDAP_UID__: The UID attribute for entries in the LDAP server. Default is `uid`.
* __WEB_LDAP_FILTER__: The optional filter to use for LDAP user searching. No default value.

### Initial setup parameters

#### Jabber support
* __JABBER_USER__: The jabber user to use for notifications. No default value.
* __JABBER_PASS__: The jabber password associated with the JABBER_USER. No default value.
* __JABBER_HOST__: The jabber host to connect to. Defaults to the fqdn after the @ symbol of the jabber user. For example a JABBER_USER of `person@im.corp.com` would have a deafult of `im.corp.com`.
* __JABBER_PORT__: The jabber port to connect to. Default is `5222`.

#### Thruk user configuration
* __WEB_ADMIN_PASSWORD__: The password to use for the thrukadmin user. The default is a randomly generated password that will be output on the command line at initial setup.
* __WEB_USERS_FULL_ACCESS__: Allow all authenticated users full access to the Web UI monitoring. Useful for situations where the `WEB_LDAP_FILTER` already restricts access to users with specific attributes. Default `false`.

#### Email notification configuration
* __NOTIFICATION_FROM__: The "from" address for naemon notifications. Default is `naemon@$HOSTNAME`. _Note: This is changeable in the externalized configuration.

## Known Issues / Warnings

##### Plugin symlinks in the /data volume are not resolvable outside of the container

The default naemon plugins are installed with the OS and resolve to a variety of places within the container file system. This may make for a very confusing situation if trying to chase down files outside of the container.

## Contributing

Contributions and feedback are welcome. Please file issues using the github project's issue tracker and send pull requests through github.