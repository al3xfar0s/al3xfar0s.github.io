---
layout: post
title: Pi-hole and Docker - How to enable HTTPS
date: 2021-01-12 13:00:00 +0100
description: Quick guide on how to enable HTTPS on a docker running Pi-hole # Add post description (optional)
img: pihole+docker.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [Pi-hole, Docker, HTTPS, SSL, letsencrypt]
---

Pi-hole is a Linux network-level advertisement and Internet tracker blocking application which acts as a DNS sinkhole and optionally a DHCP server, intended for use on a private network. Spinning it up in Docker is quite fast and easy.

Unfortunately, the [official documentation](https://github.com/pi-hole/docker-pi-hole) does not provide the steps needed to enable HTTPS on a Docker based Pi-hole instance and lots of users find this tricky. If you want to enable HTTPS on Pi-hole then this is your guide.

First of all, I'm assuming that you have already setup Let's Enrypt and you have your SSL cert handy. I'm not going to go through that process as there are hundreds, if not thousands guides out there.

Now, first thing we have to do is to create a directory where you will save your config and your pem files. I have created the directory in a NAS drive for example, where I have all the config files of my docker images.

{% highlight bash %}
mkdir /srv/sda/Config/pihole
{% endhighlight %}

Make sure the owner is set to your docker user. If your docker user is dockeruser then run:

{% highlight bash %}
chown -R dockeruser:users /srv/sda/Config/pihole
{% endhighlight %}

Next step, let's copy the files over and make sure we have the proper permissions set. Create upload-ssl-pi-hole.bash in your home dir and paste the below. Make sure to change yoursite.xyz to your website and also the destination path:

{% highlight bash %}
sudo cat /etc/letsencrypt/live/yoursite.xyz/privkey.pem /etc/letsencrypt/live/yoursite.xyz/cert.pem | sudo tee /etc/letsencrypt/live/yoursite.xyz/combined.pem
sudo cp /etc/letsencrypt/live/yoursite.xyz/combined.pem /srv/sda/Config/pi-hole/lighttpd/combined.pem
sudo cp /etc/letsencrypt/live/yoursite.xyz/fullchain.pem /srv/sda/Config/pi-hole/lighttpd/fullchain.pem
sudo chown www-data:users /srv/sda/Config/pi-hole/lighttpd/combined.pem
sudo chown www-data:users /srv/sda/Config/pi-hole/lighttpd/fullchain.pem
sudo docker restart pihole
{% endhighlight %}

The above has to run every time we generate a new certificate, let's create a cronjob and schedule it to run after your Let's Encrypt cronjob runs:

{% highlight bash %}
30 5 1 * * /home/dummyuser/upload-ssl-pi-hole.bash
{% endhighlight %}

In my case, Let's encrypt runs on the 1st of every month at 5am and I have scheduled the above job to run at 5.30am.

Another file we have to create is external.conf. Make sure you are in the lighttpd folder(/srv/sda/Config/pi-hole/lighttpd in this example), run vim external.conf and paste the below. Make sure to change the FQDN to your domain name:

{% highlight bash %}
$HTTP["host"] == "yoursite.xyz" {
  # Ensure the Pi-hole Block Page knows that this is not a blocked domain
  setenv.add-environment = ("fqdn" => "true")

  # Enable the SSL engine with a LE cert, only for this specific host
  $SERVER["socket"] == ":443" {
    ssl.engine = "enable"
    ssl.pemfile = "/etc/lighttpd/combined.pem"
    ssl.ca-file =  "/etc/lighttpd/fullchain.pem"
    ssl.honor-cipher-order = "enable"
    ssl.cipher-list = "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH"
    ssl.use-sslv2 = "disable"
    ssl.use-sslv3 = "disable"
  }

  # Redirect HTTP to HTTPS
  $HTTP["scheme"] == "http" {
    $HTTP["host"] =~ ".*" {
      url.redirect = (".*" => "https://%0$0")
    }
  }
}
{% endhighlight %}

Now that we have all the above in place we can go through the docker compose file, the official docker compose file has 2 folders that must be mounted locally in order to preserve settings, /etc/pihole/ and /etc/dnsmasq.d/

We need to mount 3 more files in order to make this work, combined.pem, fullchain.pem and external.conf. 

Below is an example of the docker compose file I am using to make this work. Again, make sure the paths are up to date and the password is set to your password. If you want to use the random password feature then comment it out.

{% highlight bash %}
version: "2"

# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "80:80/tcp"
      - "443:443/tcp"
    environment:
      TZ: 'Europe/London'
      WEBPASSWORD: 'yourpassword'
    volumes:
      - '/srv/sda/Config/pi-hole/etc-pihole/:/etc/pihole/'
      - '/srv/sda/Config/pi-hole/etc-dnsmasq.d/:/etc/dnsmasq.d/'
      - '/srv/sda/Config/pi-hole/lighttpd/combined.pem:/etc/lighttpd/combined.pem'
      - '/srv/sda/Config/pi-hole/lighttpd/fullchain.pem:/etc/lighttpd/fullchain.pem'
      - '/srv/sda/Config/pi-hole/lighttpd/external.conf:/etc/lighttpd/external.conf'
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
{% endhighlight %}

And this is it, you now have Pi-hole running in Docker on HTTPS!

P.S. make sure you run once manually the upload-ssl-pi-hole.bash **before** you start your docker image, otherwise it will fail since the certs and the external.conf file won't be there.
