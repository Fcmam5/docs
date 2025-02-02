---
slug: monitor-services-with-nagios-on-debian-5-lenny
deprecated: true
author:
  name: Linode
  email: docs@linode.com
description: 'Use Nagios to monitor services and send status updates on your Debian 5 (Lenny).'
keywords: ["nagios", "monitoring"]
tags: ["debian","monitoring"]
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
aliases: ['/server-monitoring/nagios/debian-5-lenny/','/uptime/monitoring/monitor-services-with-nagios-on-debian-5-lenny/']
modified: 2011-06-30
modified_by:
  name: Linode
published: 2010-04-21
title: 'Monitor Services with Nagios on Debian 5 (Lenny)'
relations:
    platform:
        key: install-nagios-monitoring
        keywords:
            - distribution: Debian 5
---

Nagios is an open source monitoring tool that monitors services within a server cluster. In addition to monitoring services such as SMTP/POP3 and HTTP, Nagios can also be used to check the health of the server itself.

Before continuing with this guide, you will need to make sure that you have set your hostname by following the steps outlined in the [Setting Up and Securing a Compute Instance](/docs/guides/set-up-and-secure/). Additionally, you will also need to have a functioning [LAMP stack](/docs/guides/lamp-server-on-debian-5-lenny/) in order to use Nagios.

## Prepare for Nagios Installation

Issue the following commands to update your system's package repositories and installed packages as well as install dependencies for Nagios:

    apt-get update
    apt-get upgrade
    apt-get install build-essential php5-gd wget libgd2-xpm libgd2-xpm-dev

You will also need to create a new system user account to run Nagios under. Issue the following commands to create the `nagios` user and `nagcmd` group:

    adduser --system --no-create-home --disabled-login --group nagios
    groupadd nagcmd
    usermod -G nagcmd nagios
    usermod -a -G nagcmd www-data

## Install Nagios

### Download Nagios Software

Begin by downloading the latest version of the Nagios Core package. At the time of writing the latest version of Nagios is 3.2.3, but be sure that you are downloading the latest version by checking the [Nagios download page](http://www.nagios.org/download/core/thanks/). Issue the following commands:

    cd /opt/
    wget http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-3.2.3.tar.gz

Next, download the official plugin package for Nagios. At the time of writing this is version 1.4.15, but check the [Nagios plugin download page](http://www.nagios.org/download/plugins/) to verify.:

    cd /opt/
    wget http://prdownloads.sourceforge.net/sourceforge/nagiosplug/nagios-plugins-1.4.15.tar.gz

### Build and Install Nagios Core

Issue the following commands to extract, compile, and install the Nagios Core:

    tar -xzvf /opt/nagios-3.2.3.tar.gz
    cd /opt/nagios-3.2.3
    ./configure --with-command-group=nagcmd
    make all
    make install
    make install-init
    make install-config
    make install-commandmode

### Configure Nagios Core

Begin by editing the `/usr/local/nagios/etc/objects/contacts.cfg` file's email field according to the example below:

{{< file "/usr/local/nagios/etc/objects/contacts.cfg" >}}
define contact{
    contact_name nagiosadmin ; Short name of user use generic-contact
    ; Inherit default values from generic-contact template (defined above)
    alias John Doe ; Full name of user
    email <someadmin@example.com> ; <<***** CHANGE THIS TO YOUR EMAIL ADDRESS ******
}

{{< /file >}}


Issue the following commands to configure the web interface for Nagios:

    cd /opt/nagios-3.2.3
    make install-webconf

You will now need to create a password for the `nagiosadmin` user. You will use this password to log into the Nagios administration panel when it is configured.

    htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

In order for your configuration changes to be applied, you will need to reload the web server.:

    /etc/init.d/apache2 reload

### Build and install Nagios Plugins

To install and configure the Nagios plugins, issue the following commands:

    cd /opt/
    tar xzf /opt/nagios-plugins-1.4.15.tar.gz
    cd /opt/nagios-plugins-1.4.15
    ./configure --with-nagios-user=nagios --with-nagios-group=nagios
    make
    make install

You're now ready to run Nagios for the first time and continue with the configuration process!

## Running Nagios

Issue the following commands to ensure that Nagios is started when your system boots:

    chmod +x /etc/init.d/nagios
    /usr/sbin/update-rc.d -f nagios defaults

Run the following command to check your Nagios configuration file for errors:

    /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg

Any errors will be shown in red. If everything is okay, you may issue the following command to start Nagios for the first time:

    /etc/init.d/nagios start

You may now access the web based administration and reporting tools by visiting `http://example.com/nagios/`, where `example.com` refers to your Linode's default virtual host. You may also access this interface by visiting `http://12.34.56.78/nagios/` where `12.34.56.78` is the IP address of your Linode. You will need to authenticate with the nagiosadmin user you created earlier.

**Please note:** The above example does not use SSL and your password will be sent unencrypted. You will need to generate an SSL certificate and install it yourself. Steps for doing so can be found in our [SSL guide](/docs/guides/create-a-self-signed-tls-certificate/).

## Configure Nagios Alerts

A great deal of the power of Nagios is its ability to send notifications and alerts regarding the status of services and devices. While most of this fine-grained configuration is beyond the scope of this document, we have outlined some basic notifications below.

### Installing Prerequisites to send Mail Alerts

Before Nagios can send alerts by email, basic mail services need to be installed. Issue the following command:

    apt-get install mailutils postfix

When the installation process prompts you to define the type of mail setup you're running, select "Internet Site". You will also want to specify the machine specific hostname for this server during the installation process. Next, you'll need to update the path to the mail binary in the Nagios command file. Change both references from `/bin/mail` to `/usr/bin/mail`. The relevant section of this file should look like this:

{{< file "/usr/local/nagios/etc/objects/commands.cfg" >}}
define command{
    command_name    notify-host-by-email
    command_line    /usr/bin/printf "%b" "***** Nagios *****\n\nNotification Type: $NOTIFICATIONTYPE$\nHost: $HOSTNAME$\nState: $HOSTSTATE$\nAddress: $HOSTADDRESS$\nInfo: $HOSTOUTPUT$\n\nDate/Time: $LONGDATETIME$\n" | /usr/bin/mail -s "** $NOTIFICATIONTYPE$ Host Alert: $HOSTNAME$ is $HOSTSTATE$ **" $CONTACTEMAIL$
}

# 'notify-service-by-email' command definition
define command{
    command_name    notify-service-by-email
    command_line    /usr/bin/printf "%b" "***** Nagios *****\n\nNotification Type: $NOTIFICATIONTYPE$\n\nService: $SERVICEDESC$\nHost: $HOSTALIAS$\nAddress: $HOSTADDRESS$\nState: $SERVICESTATE$\n\nDate/Time: $LONGDATETIME$\n\nAdditional Info:\n\n$SERVICEOUTPUT$" | /usr/bin/mail -s "** $NOTIFICATIONTYPE$ Service Alert: $HOSTALIAS$/$SERVICEDESC$ is $SERVICESTATE$ **" $CONTACTEMAIL$
}

{{< /file >}}


In order for these changes to take effect, you will need to restart Nagios:

    /etc/init.d/nagios restart

### Configuring Basic IRC Notifications

You can also configure Nagios to send notifications to an IRC channel through a bot. Issue the following commands to download, extract, and build the IRC bot:

    cd /opt/
    wget http://www.vanheusden.com/nagircbot/nagircbot-0.0.29b.tgz
    tar -zxvf /opt/nagircbot-0.0.29b.tgz
    cd /opt/nagircbot-0.0.29b
    make
    make install

You will need to provide a few parameters to the IRC bot such as its nickname and server in order to use it. For a list of parameters, issue the following command:

    nagircbot -h

To start the Nagios IRC bot, issue the `nagircbot` command. See the following example as a starting point. Replace `irc.example.net` with the address of your IRC network, and `#example` with the name of the IRC channel that the bot should send notifications to. Replace `nagircbot` with the desired nickname for your bot, and `ident` and `realname` with the ident and real name that the IRC bot should use.

    nagircbot -f /usr/local/nagios/var/status.dat -s irc.example.net:6667 -c \#example -C -n nagircbot -u ident -U realname -I 900

In this example, "-f /usr/local/nagios/var/status.dat" tells the bot where to get status updates. The "-C" flag will allow the bot to send colored messages to the channel depending on the status of the service. When services are down, red status messages are displayed. These messages turn green when the service has recovered. Warnings are displayed in yellow, but do not typically represent a critical issue. The "-I 900" parameter tells the bot to send a status message to the channel every 900 seconds (or 15 minutes). For example, the bot may send something like "Critical: 0, warning: 1, ok: 6, up: 2, down: 0, unreachable: 0, pending: 0", which indicates that there are no critical messages and 1 warning.

Nagios contains numerous features that are beyond the scope of this document. You are encouraged to explore the resources listed below and the administrative interface for more information regarding the setup and configuration of Nagios.

## More Information

You may wish to consult the following resources for additional information on this topic. While these are provided in the hope that they will be useful, please note that we cannot vouch for the accuracy or timeliness of externally hosted materials.

- [Nagios Home Page](http://www.nagios.org/)
- [Nagios IRC Bot](http://exchange.nagios.org/directory/Addons/Notifications/IRC/nagircbot/details)
- [Nagios Library](http://library.nagios.com/)
- [Nagios Security](http://nagios.sourceforge.net/docs/3_0/cgisecurity.html)



