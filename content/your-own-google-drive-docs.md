---
title: "Make your own Google Drive+Docs with Nextcloud, Collabora Online and object storage"
description: As someone who values privacy (mine and others'), I usually try to find new ways of getting rid of the now infamous GAFAM and their friends, the biggest of them all being Google. Among every Google service, there's one that is hugely used among both individuals and organisations, which is Google Docs. Add to that the whole storage service they also provide, and you get Google Drive, the best way to directly feed Google with all your files and data, including personal and administrative documents, music collections, photos from your smartphone... Because I don't want to share that huge amount of data with Google, I've set up my own Google Drive+Docs using mainly self-hosted free software projects, including Nextcloud, Caddy and (Docker-less) Collabora Online.
tags: [ 'google', 'gafam', 'floss', 'nextcloud', 'collabora', 'dockerless', 'openstack', 'swift' ]
publishDate: 2018-07-13T00:00:00+02:00
draft: false
thumbnail: /your-own-google-drive-docs/collabora.png
---

As someone who values privacy (mine and others'), I usually try to find new ways of getting rid of the now infamous GAFAM and their friends, the biggest of them all being Google. Among every Google service, there's one that is hugely used among both individuals and organisations, which is Google Docs. Add to that the whole storage service they also provide, and you get Google Drive, the best way to directly feed Google with all your files and data, including personal and administrative documents, music collections, photos from your smartphone... In other words, a huge volume of data that either is litteraly personal data, or can be used to extract personal data about you without your explicit consent (in Google's case).

In my case, I'm not really at ease with the fact that I need to upload administrative documents containing personal data to Google in order to register for my boat driving license, nor do I with the fact that my phone will automatically upload every picture I take to Google's servers so it can be processed and have any data it contains extracted and stored in a database I don't have control over.

Just as many reasons for me to look for a Google Drive-like solution that I could entirely control. Here's what I came up with so far:

* [Nexctloud](https://nextcloud.com/) 13 as the whole cloud management solution
* [OpenStack](https://www.openstack.org/)'s object storage (Swift) as the scalable storage backend
* [Caddy](https://caddyserver.com/) as the web server
* [Collabora Online](https://www.collaboraoffice.com/code/) (without Docker) as the collaborative office suite

So far I got all of that working together on my personal space, so let me walk you through how to do that yourself. Because of the whole stack's current state, it does require some technical skills, though. Sorry about that.

***Update:** right after I published this post, [Antoine](https://twitter.com/nd4pa) made an amazing Ansible playbook based on it, which you can find [right here](https://github.com/nd4pa/homesuite-ansible)!*

*(Disclaimer: because I know that some who read this blog might ask the question, let me get things straight first: yup, I'm using Nextcloud as my Google Drive replacement, even though I currently work at [CozyCloud](https://cozy.io). Although people can seem to think it might be hypocritical from me to do so, my take on the matter is that Cozy and Nextcloud aren't competitors, and although they have some features in common, one can easily complete the other, as Cozy (which I also use) has features Nexctloud doesn't have and vice-versa. Diversity is great, as people from both [CozyCloud](https://twitter.com/cozycloud/status/968763001540145152) and [Nextcloud](https://twitter.com/Nextclouders/status/978233603191603200) would tell you. And no, I haven't been asked to write that paragraph 😉)*


## LCPP: (GNU/)Linux-Caddy-PostgreSQL-PHP

### Get the right hosting

First things first, because I want to have control over the whole thing, I'm self-hosting most of the pieces of software I previously mentioned on a personal VPS. Mine is a [START 1-S from Scaleway](https://www.scaleway.com/pricing/), because, since the storage backend will be remote (I'll come back to that later on), I needed a better bandwidth than the best-effort 100Mbps one I have on my OVH cloud instances (and I know that because I previously tried this setup on one of these, which resulted in poor performances), so mine has a 200Mbps bandwidth (I couldn't find any piece of info about whether that was guaranteed or best-effort, though).

For the record, OVH does offer instances with guaranteed 250Mbps bandwidth, though given their rates it's not something I can afford only for that use, especially given what competitors offer, including Scaleway.

Anyway, try to keep the bandwidth requirement in mind while chosing where you'll host your Nextcloud instance.

Also for the record, the operating system on my VPS at the time of writing is GNU/Linux, more precisely Ubuntu 16.04 LTS, so the processes I'll describe in this blog post might slightly differ if you're running another GNU/Linux distribution or another operating system.

### Caddy

As I mentioned, I'm using [Caddy](https://caddyserver.com) as the web server for the whole thing. In case the name doesn't ring a bell, Caddy is a lightweight web server with simple configuration, HTTP/2 as its default, automatic HTTPS through Let's Encrypt, and a lot of available plugins.

One downside of using Caddy is that, unlike popular free software projects out there, it doesn't provide any official packaging (because of complications brought by the plugins system), so installation must be done manually. This is not that hard, though, because the whole process is pretty much straightforward:

{{< highlight bash >}}
# Download caddy
curl -L "https://caddyserver.com/download/linux/amd64?license=personal&telemetry=off" > caddy.tar.gz

# Extract the whole thing in a specific directory
mkdir caddy
cd caddy
tar xvf ../caddy.tar.gz

# Install Caddy's binary and allow it to use the 80 and 443 ports
sudo cp ./caddy /usr/local/bin/
sudo setcap cap_net_bind_service=+ep /usr/local/bin/caddy

# Create Caddy's certificates directory
sudo mkdir /etc/ssl/caddy
sudo chown -R www-data:www-data /etc/ssl/caddy

# Create Caddy's configuration directory
sudo mkdir /etc/caddy /etc/caddy/caddy.conf.d
sudo echo "import caddy.conf.d/*.conf" > /etc/caddy/Caddyfile
sudo chown -R www-data:www-data /etc/caddy

# Create, enable and start Caddy's systemd service
sudo mkdir /usr/lib/systemd/system
sudo cp ./init/linux-systemd/caddy.service /usr/lib/systemd/system
sudo systemctl enable caddy
sudo systemctl start caddy
{{< / highlight >}}

Now that you have Caddy installed, let's install another very important component we'll need to run Nextcloud: PHP.

### PHP

Because Caddy can only interact with PHP using FastCGI (as far as I'm aware of), we'll need to install its *FPM* (FastCGI Process Manager) version, named `php-fpm`. Along with that, you'll need a fair amount of PHP extensions Nextcloud depends on. We'll also be using PostgreSQL to manage Nextcloud's database, so we'll also need drivers for that.

On most Debian-based systems, this is done by doing the following:

```
sudo apt install php7.0-fpm php7.0-gd php7.0-json php7.0-pgsql php7.0-curl php7.0-mbstring php7.0-intl php7.0-mcrypt php-imagick php7.0-xml php7.0-zip
```

PHP's FPM should now be installed, configured with the right extensions, and running. You can make sure of that by checking whether the file `/var/run/php/php7.0-fpm.sock` exists.

In order to be sure that all PHP extensions we just installed are loaded, let's restart its FPM:

```
sudo systemctl restart php7.0-fpm
```

### PostgreSQL

Nextcloud needs a database, so let's install a databases management system! I personally use PostgreSQL because of its performances and ease to use. You can install it by simply running:

```
sudo apt install postgresql
```

Now, we need to do some basic configuration within PostgreSQL so Nextcloud has a user and a database to connect to. PostgreSQL provides an interactive shell (`/usr/bin/psql`), with the user `postgres` acting as a superadmin by default.

Regarding authentication, and more precisely access to the shell from the local host, PostgreSQL default to using its `peer` authentication method, which means you can only authenticate and access the shell as a user if you're trying from a system user with the same name. As the `postgresql` package has already created a `postgres` system user, all that's left to do is to call the PostgreSQL shell as that user:

```
sudo -u postgres psql
```

*Note: if that doesn't work, it might be because your system user lacks privileges. Try that again as a user with more privileges (and/or root).*

Now create Nextcloud's user and database from that shell using SQL queries:

```
postgres=# CREATE USER nextcloud WITH password 'ncpassword';
CREATE ROLE
postgres=# CREATE DATABASE nextcloud WITH owner 'nextcloud';
CREATE DATABASE
postgres=# \q
```

In PostgreSQL's shell, `\q` is the exit instruction.

Don't forget to change `ncpassword` in my example above with a real, strong password for Nextcloud's database user. Also, keep that password somewhere, because you'll need it when installing Nextcloud. And while we're at it...

## Installing Nextcloud

Because Nextcloud is only a PHP web app, installing it is as simple as downloading a ZIP archive (you might need to run `sudo apt install unzip` in order to be able to open the archive). The link of the archive to download can be found [here](https://nextcloud.com/install/#instructions-server) as the target of the big blue "Download Nextcloud" button.

As an example, here's how your install should go with Nextcloud 13.0.4, authenticated as root:

{{< highlight bash >}}
# Change /srv/http with your web server root (e.g. /var/www)
cd /srv/http

# Download Nextcloud's archive
curl -LO https://download.nextcloud.com/server/releases/nextcloud-13.0.4.zip

# Extract the archive's content
unzip nextcloud-13.0.4.zip

# Make the Caddy user owner of the extracted directory
chown -R www-data:www-data nextcloud
{{< / highlight >}}

And here you go, with a brand new Nextcloud install located at `/srv/http/nextcloud`.

Now let's create Nextcloud's configuration file in Caddy's configuration folder. You must first have a domain, or sub-domain, Nextcloud should be accessible at (i.e. `cloud.example.tld`). Here's the content of my own `/etc/caddy/caddy.conf.d/nextcloud.conf` file:

```
cloud.example.tld {
	tls letsencrypt@example.tld

	root   /srv/http/nextcloud

	fastcgi / /var/run/php/php7.0-fpm.sock php {
		env PATH /bin
	}

	# checks for images
	rewrite {
		ext .svg .gif .png .html .ttf .woff .ico .jpg .jpeg
		r ^/index.php/(.+)$
		to /{1} /index.php?{1}
	}

	rewrite {
		r ^/index.php/.*$
		to /index.php?{query}
	}

	# client support (e.g. os x calendar / contacts)
	redir /.well-known/carddav /remote.php/carddav 301
	redir /.well-known/caldav /remote.php/caldav 301

	# remove trailing / as it causes errors with php-fpm
	rewrite {
		r ^/remote.php/(webdav|caldav|carddav|dav)(\/?)$
		to /remote.php/{1}
	}

	rewrite {
		r ^/remote.php/(webdav|caldav|carddav|dav)/(.+?)(\/?)$
		to /remote.php/{1}/{2}
	}

	rewrite {
		r ^/public.php/(dav|webdav|caldav|carddav)(\/?)$
		to /public.php/{1}
	}

	rewrite {
		r ^/public.php/(dav|webdav|caldav|carddav)/(.+)(\/?)$
		to /public.php/{1}/{2}
	}

	# .htaccess / data / config / ... shouldn't be accessible from outside
	status 403 {
		/.htacces
		/data
		/config
		/db_structure
		/.xml
		/README
	}

	header / Strict-Transport-Security "max-age=31536000;"
}
```

This file is directly adapted from one of Caddy's example configuration files, and can be found [right here](https://github.com/caddyserver/examples/blob/master/nextcloud/Caddyfile). I made a few changes, which are the domain I gave Nextcloud on the first line, and the email address Let's Encrypt should send me certificates expiration notices on the second line. I also removed the logging (basically because I don't want it nor need it, since it's my personal instance) and moved the vhost's root to some other place on the disk since I want my web server's root at `/srv/http` rather than `/var/www` (but that doesn't really matter).

Now let's reload Caddy's configuration by running `sudo systemctl reload caddy`. Your Nextcloud should now be accessible from the domain (or sub-domain) you gave it, and opening that in a browser should display a configuration wizard. It can take some time, though, as Caddy needs to talk with Let's Encrypt to generate the required certificates since it's the first time it encounters this domain.

![Nextcloud setup wizard](/images/your-own-google-drive-docs/nc-wizard.png)

Within this wizard, you can provide the login and password for your administrator account. Before clicking "Finish setup", click "Storage & database" (if you can't see any other field than the ones requesting the administrator's credentials), because we need to configure Nextcloud's access to PostgreSQL.

Leave the "Data folder" field as it is, and fill in the user and password of Nextcloud's PostgreSQL user (which, in our example, should be "nextcloud" and the password you previously set, respectively), and Nextcloud's database (which, in our example, should be "nextcloud"). Because you might already have a MySQL/MariaDB/SQLite PHP driver installed, Nextcloud might offer you different databases management systems; select "PostgreSQL".

Now you can click "Finish setup", and here you have a working Nextcloud instance waiting for you!

*Note that Nextcloud also provides a way to do the whole setup process with a command line interface, as documented [here](https://docs.nextcloud.com/server/13/admin_manual/installation/command_line_installation.html).*

If you just want a simple Nextcloud instance with local storage, you can stop here. I chose to use OpenStack Swift as Nextcloud's primary storage backend, so if that's what you're looking for, don't upload anything yet and bear with me as I walk you through that setup!


## OpenStack Swift as primary storage backend

As I want to store music and videos in Nextcloud, along with backups from my phone's camera, I might get quite limited with the local disk space on my VPS. As disk space storage can get quite expensive, and is usually really limited on cheap VPSs. A great solution to that problem is cloud-based [object storage](https://en.wikipedia.org/wiki/Object_storage). The main advantage of such a solution is being a very unexpensive storage solution (usually around 0.01€/GB/month) that scales almost infinitely.

There are two main solutions of the sort, as far as I'm aware of, which are Amazon S3, and OVH's Public Cloud Storage, which is based on OpenStack Swift. Because I don't want any of my data on Amazon's servers (for the same reason I don't want it on Google's), and because I usually trust OVH with my data (mainly because I know a few people there, thus have a small peek at what internal use is done with that same data), I decided to go with OVH's solution.

Before I go any further, I'll assume you already have an OVH account (which you can create on their website). Direct your browser to OVH's [Cloud manager](https://www.ovh.com/manager/cloud), then order a new "Cloud project" using the blue "Order" button on the top left corner of the page. Once your project is created, it should appear under "Servers" in the navigation bar on the left (which you might need to click on to uncover the list of projects), click on its name, then on the "Storage" item that just appeared. Click the white "Create a container" button, which takes you to an interface letting you create an object storage container.

![Container creation interface](/images/your-own-google-drive-docs/ovh-create-container.png)

Select the datacentre that best fit your use (i.e. the one located the closest to you). Also make sure your container's type is set to "Private" so the container can't be accessed without authentication. Name your container and confirm the creation.

Now that you created your container, you'll need Nextcloud to be able to access it. First, you need to retrieve OpenStack credentials for your project. In order to get that, click the "OpenStack" entry corresponding to your project in the navigation bar on the left of the screen, then "Add user" and input a description (e.g. "nextcloud"). Copy the user's password somewhere (because OVH's manager will never displayed it again), then click on the "..." button on the right of the user's row, then "Downloading an Openstack configuration file", and select the datacentre you chose for your object storage container.

![OpenStack user](/images/your-own-google-drive-docs/ovh-openstack-user.png)

This will download a file named `openrc.sh` containing all remaining pieces of information required for Nextcloud to access your container. Now let's give it these data, by editing Nextcloud's configuration file (`/srv/http/nextcloud/config/config.php` in my case) and adding this block to the config (make sure to keep the last line (which should just be `);`) at the very end of the file in order to keep the file's syntax correct):

```
'objectstore' => array(
	'class' => 'OC\\Files\\ObjectStore\\Swift',
	'arguments' => array(
		'username' => 'OS_USERNAME',
		'password' => 'OS_PASSWORD',
		'bucket' => 'nextcloud',
		'autocreate' => false,
		'region' => 'OS_REGION_NAME',
		'url' => 'https://auth.cloud.ovh.net/v2.0',
		'tenantName' => 'OS_TENANT_NAME',
		'serviceName' => 'swift',
	),
),
```

With:

* `OS_USERNAME` being the username (or "ID", as OVH's manager calls it) of your OpenStack user.
* `OS_PASSWORD` being the password of your OpenStack user.
* `OS_REGION_NAME` being the datacentre's identifier, which would be `GRA3` in the example we've seen before example. This information can be found at the very last line from the `openrc.sh` file.
* `OS_TENANT_NAME` being the name of the OpenStack tenant to use, which can be found in the `openrc.sh` file on the line starting with `export OS_TENANT_NAME`.

Now you can refresh Nextcloud's tab in your browser, and, if everything went well, it should display Nextcloud's interface. Because the storage backend is now remote, it might be slow to display completely, though, let's see how we can improve performances using caching.

## Caching, caching everywhere

All of these solutions work pretty well together, and are part of recommendations made by Nextcloud regarding caching.

### Zend OPCache

PHP already comes bundled with a cache mechanism, which is a PHP opcache named Zend OPCache. Basically, a PHP opcache stores compiled PHP scripts so they don’t need to be re-compiled every time they are called. To enable it and get it to match Nextcloud's recommendations, uncomment the following lines and ajust the necessary values in yout `/etc/php/7.0/fpm/php.ini` file in this way:

```
opcache.enable=1
opcache.enable_cli=1
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.revalidate_freq=1
opcache.save_comments=1
```

Then all you need to do is restart PHP's FPM in order for the changes to be applied:

```
sudo systemctl restart php7.0-fpm
```

### The APCu PHP extension

Now we need to add some cache for Nextcloud's data. The easiest way to achieve that is by installing the [APCu](https://pecl.php.net/package/APCu) PHP extension:

```
sudo apt install php-apcu
```

This package automatically installs and configures the extension, so all that's left to do is restart PHP for this extension to be loaded:

```
sudo systemctl restart php7.0-fpm
```

Now let's tell Nextcloud to use APCu for local cache by adding this line to its configuration file (`/srv/http/nextcloud/config/config.php` in my case):

```
'memcache.local' => '\OC\Memcache\APCu',
```

Again, make sure to keep the file's last line (which should just be `);`) at the very end in order to keep the file's syntax correct.

### Redis

With its default configuration, Nextcloud can run into some troubles handling file locks. After switching my instance's storage backend to OVH's Public Cloud Storage, I had quite a huge volume of data to re-send from my disk to my instance, which sometimes would result in Nextcloud not being able to upload some files because of (probably) errored file locks.

A single Redis instance can act as a great cache for file locks, and installing it is as simple as running:

```
sudo systemctl restart redis-server php-redis
```

Once again, restart PHP's FPM:

```
sudo systemctl restart php7.0-fpm
```

Then let's tell Nextcloud to use Redis to cache file locks (and how to reach the Redis instance) by adding this block to Nextcloud's configuration file:

```
'memcache.locking' => '\OC\Memcache\Redis',
'redis' => array(
	 'host' => 'localhost',
	 'port' => 6379,
),
```

Again, watch out for the file's last line, you know how it goes by now 😄

Refresh Nextcloud's tab in your browser, and it should all run much faster from now on. You can also safely and efficiently start synchronising your data using Nextcloud's or ownCloud's client (since the APIs are the same between both systems).

## Collabora Online, Docker-less

Now let's talk about what took me the longest to figure out: Collabora Online (even though once you do, it's pretty simple, so it shouldn't take too long to get you set up with it).

[Collabora Online](https://www.collaboraoffice.com/code/) is a [FLOSS](https://gerrit.libreoffice.org/gitweb?p=online.git) online collaborative office suite based on [LibreOffice](https://www.libreoffice.org/). It makes an amazing replacement to Google Docs's text documents and spreadsheets editors and Nextcloud even provides an integration for it.

One of my biggest troubles, though, was that the current recommended way to install Collabora Online was through Docker. I'm personally not a huge fan of Docker, and find it has some awful design flaws when it comes to resources management.

Unfortunately, although it's possible to install Collabora Online from Collabora's Debian/Ubuntu/CentOS/openSUSE repositories, the process of setting it up is barely documented (and even not at all for some parts). After several hours of searching and experimenting with it, I finally managed to get it to work, so here's an attempt at documenting the whole thing.

The first step to take is to install Collabora's repository and the required packages. Please bear in mind that the host is running Ubuntu 16.04, so the process might differ if you're running another GNU/Linux distribution. Instructions for all supported distributions can be found [here](https://www.collaboraoffice.com/code/#packages_for_linux_x86_64_platform).

{{< highlight bash >}}
# Install support for HTTPS APT repositories
sudo apt install apt-transport-https

# Import Collabora's signing key
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 0C54D189F4BA284D

# Add the URL for the Collabora Online's repository to /etc/apt/sources.list
sudo echo 'deb https://www.collaboraoffice.com/repos/CollaboraOnline/CODE ./' >> /etc/apt/sources.list

# Perform the installation
sudo apt update && sudo apt install loolwsd code-brand
{{< / highlight >}}

Now edit (as root) the `/etc/loolwsd/loolwsd.xml` file, such that:

* in the `ssl` section, `enable` should have `false` as its value, and `termination` must be set to `true`. Because we'll serve Collabora Online behind Caddy, acting as a HTTPS reverse proxy, we don't want Collabora Online to serve its content using HTTPS (which might cause some troubles with certificates, as Caddy's way to handle these wouldn't let another program access them), but we want it to tell its clients to use HTTPS URLs instead of HTTP ones, which is exactly what `termination` does.
* in the `wopi` section (itself being located under the `storage` section), add a `host` line containing the domain name you gave Nextcloud, with dots (`.`) escaped with backslashes (`\`) because the value is expected to be a regular expression. Make sure the `allow` attribute is set to `true`. The `storage` section contains access control lists (*ACLs*) to tell Collabora Online where it can access files and where it can't. Because Nextcloud uses [WOPI](https://wopi.readthedocs.io/en/latest/) to that end, this line grants access to Nextcloud to provide storage for Collabora Online using the WOPI protocol. As an example, if Nextcloud was served at `cloud.example.tld`, the line would look like:

```
<host desc="Regex pattern of hostname to allow or deny." allow="true">cloud\.example\.tld</host>
```

Let's restart Collabora Online to let it know about the new configuration:

```
sudo systemctl restart loolwsd
```

Now we need to configure Caddy to act as a HTTPS reverse proxy for Collabora Online. Just like Nextcloud, you need a domain (or sub-domain) to give Collabora (here `collabora.example.tld`). Here's what I have in `/etc/caddy/caddy.conf.d/collabora.conf`:

```
collabora.example.tld {
	tls letsencrypt@example.tld

	proxy /loleaflet http://127.0.0.1:9980 {
		transparent
	}

	proxy /hosting/discovery http://127.0.0.1:9980 {
		transparent
	}

	proxy /lool http://127.0.0.1:9980 {
		transparent
		websocket
	}

	header / {
		Strict-Transport-Security "max-age=31536000;"
		Content-Security-Policy "default-src 'none'; frame-src 'self' blob:; connect-src 'self' wss://cloud.example.tld; script-src 'unsafe-inline' 'self'; style-src 'self' 'unsafe-inline'; font-src 'self' data:; object-src blob:; img-src 'self' data: https://cloud.example.tld:443; frame-ancestors https://cloud.example.tld:443"
	}
}
```

While the whole file is pretty basic, let's talk about the last instruction, which contains the [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy) HTTP header. Because it's a quite complex header, I don't really know the details of how to tweak it to make it work with Collabora Online, and had neither the time nor the motivation to dive into it. Therefore, the header shown here is a plain copy of the one sent by [Nextcloud's demo servers](https://demo.nextcloud.com), where I replaced every instance of `demo.nextcloud.com` with Nextcloud's URL (which, in the example shown here, is `cloud.example.tld`, which you should replace with the domain you gave Nextcloud). It works fine, but I wanted to point out that this part of the configuration isn't my own work.

Now let's reload Caddy to let it know of these changes in its configuration:

```
sudo systemctl reload caddy
```

And let's make Nextcloud talk to Collabora Online, by installing the [Collabora Online Nextcloud app](https://apps.nextcloud.com/apps/richdocuments). Documentation on installing Nextcloud apps can be found [here](https://docs.nextcloud.com/server/13/admin_manual/installation/apps_management_installation.html). Then, browse to your Nextcloud settings, and click "Collabora Online" under the administration settings section. Now all there's left to do is input your Collabora Office instance's URL (which, with the example shown here, would be `https://collabora.example.tld`), hit "Save" and here you go! Nextcloud is now correctly hooked up with Collabora Online! 😁

If you want to try it out, you can upload a `.doc`/`.docx`/`.odt`/`.ods`/etc. file to Nextcloud and open it, or create a new document using the "+" button at the top of the Files app and open it. Nextcloud should then redirect you to Collabora Online's interface, and you can start editing right away!

![Collabora Online web interface's](/images/your-own-google-drive-docs/collabora.png)

## Conclusion

Now we have a wonderful self-hosted personal Google Drive/Docs alternative only made of FLOSS projects, which is pretty neat!

However, as you might have noticed, this setup does require quite a high and broad technical knowledge to manage, which is quite sad as it makes the whole thing out of the reach of non-technical people. For some parts, the lack of documentation even makes it out of reach of some people that actually have a technical background, though there are some other ways to end up with such a setup that would require less hassle through inexisting doc (such as using Docker to run Collabora Online with, or using nginx or Apache as the HTTPS reverse proxy).

Because of all that, waving Google & friends goodbye won't be easy, if not unmanagable, for most people in the state things currently are. But to me the future isn't that dark regarding the rise of privacy-respectful alternatives, since we just saw that the tools themselves exist, and what's left to do is find a way to make them more accessible. And although this blogpost focuses entirely on the file hosting and document editing services, there's still a lot of services to build alternatives to, such as [PeerTube](https://joinpeertube.org/en/), an amazing decentralised, federated and peer-to-peer alternative to YouTube, which recently ended a very successful [crowdfunding campaign](https://www.kisskissbankbank.com/en/projects/peertube-a-free-and-federated-video-platform) at over 250% of their initial goal.

In my opinion, all that show quite a promising future for the Internet, in which anyone would at some point be able to benefit from amazing services without having to make unfair compromises such as delivering all of its personal data just to send a message to some friends. It does require some work, though, but as members of the French non-profit organisation [Framasoft](https://framasoft.org/) would say, the journey is long, but the path is free.

Aaaand that's all for this week! I hope you've enjoyed reading this post (which is now my longest one as it's slightly longer than [Enter the Matrix](/enter-the-matrix)), and if you did, or if you have any kind of feedback regarding it, please feel free to hit me up on [Twitter](https://twitter.com/BrenAbolivier), [Mastodon](https://mastodon.social/@babolivier) or [Matrix](https://matrix.to/#/@Brendan:matrix.trancendances.fr), I would love to see what you thought of it!

Regarding the rate these posts go out at, it's likely that I won't be able to carry a weekly schedule forever. This small paragraph isn't in any way the announcement of a new rate, but is rather to make it clear that I might skip a week when I deem it necessary. I'll try to keep it as close to weekly as possible, though, and whatever happens, the best way not to miss any post is to subscribe to the [RSS feed](https://brendan.abolivier.bzh/index.xml) or follow me [on Twitter](https://twitter.com/BrenAbolivier) or [Mastodon](https://mastodon.social/@babolivier) (which I'll try to share more stuff on).

Anyway, see you next week (hopefully) for a brand new blog post! 😄
