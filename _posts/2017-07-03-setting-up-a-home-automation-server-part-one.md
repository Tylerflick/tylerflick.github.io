---
layout:     post
title:      Creating a home automation system - Part One
date:       2017-07-03
summary:    Setting up the server
categories: home-automation
tags:       raspberry-pi mqtt rabbitmq home-automation
---

With the advent of the Raspberry Pi and cheap microcontroller's like the EPS8266, home automation has never been easier. This is the first installment of a number of guides I'm putting together based on setting up a home automation server, security and connecting with other services.

## The Hardware

I chose to have the Raspberry Pi 3 as the center of my system. This was for a few reasons:

- it has a global community, and this means a wealth of resources
- it's affordable
- it's plug and play (you can simply swap out SD cards to change the Pi's use case)
- it has built in WiFi and Bluetooth LE

That's not to say you can't use different hardware, but this guide is based around the Pi 3 and Raspbian.

### Setting up an Access Point

The first step in this guide is to setup the Pi's built in WiFi adapter as access point. Why? Well, it's an isssue of trust. Origally this idea was prompted by the release of the ESP8266 a few years ago. The chip is a fantastic bang for your buck microcontroller, but its SDK is still compromised of binary blobs, and I cannot leave myself to trust it.

To get around the issue of trust, I decided to use the Pi's WiFi to host a limitted connection access point, and connect to the main network via eithernet. By limiting its access only to the services and ports exposed on the Pi, I could mitigate any backdoors that may exist. (Based on the level of popularity and scrutiny that the chip has received, I highly doubt there is any malicious logic running on the chip, but you never know.)

![Connection Graphic]({{ site.url }}/images/connection_graphic.png)

**If you're not starting from a fresh install of Raspbian and don't want to lose any prior setup or configurations, I recommend backing up your Pi's disk image. This is something I do on a monthly basis.**  

Adafruit has a [nice write-up](https://learn.adafruit.com/setting-up-a-raspberry-pi-as-a-wifi-access-point/overview) on setting the Pi up as an access point, and I'm not going to regurgitate that here. The only thing to change is to omit setting up [network address translation](https://learn.adafruit.com/setting-up-a-raspberry-pi-as-a-wifi-access-point/install-software#configure-network-address-translation); you don't want to expose internet access on the limited network.

Once this is done, and the connection is verified, it's time to setup the messaging queue.

## Installing RabbitMQ + MQTT Plugin

In order to provide a generic means of communication between embedded devices, services, etc, we need a queue. Fortunatly this problem is well known and has been address by many protocols. In my opinion though, there is none better than MQTT for embedded devices.

Originally I had used [Mosquitto](http://mosquitto.org), by Roger Light, as my MQTT broker. Over time though, I ran into issues with hosting TLS connections. So I searched the internet for alternatives and ran across RabbitMQ, an AMPQ server. In a prior job I had used RabbitMQ in production and found it to be reliable so I gave it a shot. Reading through the docs I found it met all of my requirements:

- it bridges with MQTT via a plugin
- supports multiple protocals: AMPQ, STOMP, WebSockets
- has a built in web administration portal
- hosts TLS connections

### Installing RabbitMQ

RabbitMQ can be setup per the [official instructions](https://www.rabbitmq.com/install-debian.html) for Raspbian (Debian).
Just setting up via APT can be done with the following commands.

~~~bash
echo 'deb http://www.rabbitmq.com/debian/ testing main' | sudo tee /etc/apt/sources.list.d/rabbitmq.list
wget -O- https://www.rabbitmq.com/rabbitmq-release-signing-key.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install rabbitmq-server
~~~
 
### Bridging with MQTT
Now that the message broker is installed, it is time enable it to speak some MQTT. Luckily, this is just a couple file edits.

#### Enabling the plugin

1. Start by taking down the service with `sudo service rabbitmq-server stop`.
2. Change into RabbitMQ's config directory: `cd /etc/rabbitmq`.  
2. Edit enabled_plugins with whichever editor you prefer: `vim enabled_plugins`.  
3. Add or amend the following entry: `[rabbitmq_management, rabbitmq_mqtt]`.  
*Note: `rabbitmq_management` can be omitted if you don't want to expose the web management portal, but I've found it convenient for handling user management.*

#### Configuring MQTT
Setting up the plugin for non encrypted communication is simple. Staying in RabbitMQ's config directory:

1. Open the file `rabbitmq.config` for editing: `vim rabbitmq.config`
2. Replace any existing configuration with the following:

~~~erlang
[{rabbit,        [{tcp_listeners,    [5672]}]},  
 {rabbitmq_mqtt, [{default_user,     <<"guest">>},  
                 {default_pass,      <<"guest">>},
                 {allow_anonymous,   true},
                 {vhost,             <<"/">>},
                 {exchange,          <<"amq.topic">>},
                 {subscription_ttl,  1800000},
                 {prefetch,          10},
                 {ssl_listeners,     []},
                 %% Default MQTT with TLS port is 8883
                 %% {ssl_listeners,  [8883]}
                 {tcp_listeners,     [1883]},
                 {tcp_listen_options,[{backlog,   128},
                                     {nodelay,   true}]}]}
].
~~~
This sets up RabbitMQ to listen on port 1883, the default, for non TLS encrypted MQTT connections. *TLS setup will be covered later*.
3. Bring RabbitMQ back online with `sudo service rabbitmq-server start`.

## Testing out the queque
Time to talk to the message queue. Although I know longer use Mosquitto as a server, the clients are a great way to debug connection issues. You can install them with `sudo apt-get install mosquitto-clients`. Once the clients are installed we can publish a message to the server in one terminal and listen for it on another.

In the first terminal run the following command: `mosquitto_sub -h YOUR_PI_IP -p 1883 -t '/test_topic'`, replacing `YOUR_PI_IP` with whatever the IP of your server is. Enter the command and the process should begin listening to the queue.

The next terminal can now be used to send a message to the queue. Execute the following: `mosquitto_pub -h YOUR_PI_IP -p 1883 -t '/test_topic' -m 'hello world'`, again replacing `YOUR_PI_IP` with your server's IP. If everything was setup correctly you should see "hello world" in the first terminal.

RabbitMQ is now ready to begin serving up MQTT to your devices. If you install the web portal you can access it on port 15672 of your Pi's hostname.local or its IP address. From here you can add and edit users and permissions. The default admin username and password is 'guest' & 'guest'.

*Note: 'guest' access is limited to localhost.*

I'll cover setting up a TLS endpoint for RabbitMQ in the next session.