# Self-Hosted Plex CDN

So... you alike me, brave traveler have ended up here in search of a way to make your Plex Media Server instance peer better between you and your host. Worry not, we have been working on this as our pet project for months and will give you the crash course on how this fucking thing works.

## About me

I am a recent grad from Western University, where I studied Media, Information, and Technoculture; or as you might know it "Media Studies." Now I've graduated with crippling amounts of student debt, and instead of paying it all off... I ended up building a CDN and writing this guide that you're reading now... If this was a help to you, please consider [checking out my GH Sponsors Page](https://github.com/sponsors/brettpetch).

## Why the fuck would you bother?

Well... how's that Plex experience in the US from your server overseas?

Shit you say? I wonder why... Some guy I know, describes peering as follows:

> Peering issues simply mean that somewhere along the route between point A and point B, there's some congestion or fault drastically reducing the speed at which traffic can pass between both points.
> Imagine that all internet providers in the world, are trucking companies - They use highways to deliver their goods from the origin location to its destination.
>
> Some highways have 2 lanes, some have 4, some have 6 - Some are very congested on Mondays, some are not.
>
> Imagine that you live in Miami, and your ISP (Trucking company) sends their goods via the i-95 to Boston everyday with no problem, and have done so for over a year now.
>
> However, for some reason in 2019 during February, some meteor crashed into the i95 causing damage to 2 of the 4 available lanes, which in turn resulted in congestion on the i95 because the throughput is now much less then what it normally is.
>
> What we now experience is that trucks are not reaching Boston in time, they are very delayed and so the speed at which trucks were traveling from Miami to Boston was significantly reduced.
>
> Some trucking companies might re-route the trucks away from the i95, and onto regular roads, but regular roads don't have the capacity or speed limits that the highway does, so this also results in delays going from Miami to Boston, resulting in slower speeds in terms of trucks getting from A to B.
> As you can probably imagine, peering issues based on the illustration below can be sporadic - They can happen at anytime, and they can last for no more then a few hours, up to days or in really bad cases, months.

## Requirements

- An AWS account (with billing and a hosted zone), that's right you cheap fuck... Get off your wallet.
- A few servers laying around (ideally 1gbps+)
- A number of beverages (preferably beers, find a nice IPA...)

## AWS Config

Ok, you've got your billing panel setup in AWS, you're ready to get going on your first project... How do we do it?

Open up the Route 53 Dashboard. This will be responsible for the way that we send traffic around the globe. As you might know, DNS, or the Domain Name System is how your client will resolve a domain (`google.com`) to an ip address (`8.8.8.8`). Based on this, we can do a number of things, Route 53 was chosen because it enables you to do what is called Geographic DNS records, meaning you can target areas and have it resolve to a different IP based on the area reported. In our case, I'll be standing up a node in the US for our example.

First, create a hosted zone. This will cost you about $0.50 per month for your first 25 domains, then another $0.10 per additional. Note that this is per domain, not per record. From there, you are using Geo DNS and Geoproximity queries, which are $0.70 per million queries for the first billion requests per month, then $0.35 per million for anything over that. My pricing on 2 domains comes out to about $4.20/mo, but your milage may vary.

Second, we're going to configure 2 subdomains. One for each of our servers. This will be an `A` record and an `AAAA` record, which we'll make into a `CNAME` record due to the need to route to each region differently across both IPv4 and IPv6 address spaces.

From here, we're going to create the record, it will be a SIMPLE A record for `vps1.region.example.com`, here we're going to set the record to the IPv4 address of the bouncer server. Next, we'll create one for our origin, so we'll call that `srv1.example.com`, this will be our ingest for the CDN, IE the "origin". Then we'll create one more, which is a Geo record, which is called `srv1.cdn.example.com`, this will be the domain that the CDN listens on for all connections. This will resolve to a CNAME record to `vps1.region.example.com` in the region we want to bounce through your proxy, then the main will be the "default" server for other regions to hit, so a secondary CNAME record will be created pointing `srv1.cdn.example.com` at `srv1.example.com`

Basically we should have

```plaintext
vps1.sg.example.com  A      Simple       -        <YOUR_BOUNCER_IP>
srv1.example.com     A      Simple       -        <YOUR_IPv4_ADDRESS>
srv1.example.com     AAAA   Simple       -        <YOUR_IPv6_ADDRESS>
*.cdn.example.com    CNAME  Geolocation  Default  srv1.example.com
*.cdn.example.com    CNAME  Geolocation  Asia     vps1.sg.example.com
```

This configuration will resolve domains correctly. Now we're in a bit of a dilemma, how do we have nginx point between the instances correctly.

## Letsencrypt Configuration

Ok, so we've got a domain, we're going to need certs... And AWS + acme.sh has us covered. https://github.com/acmesh-official/acme.sh/wiki/dnsapi#10-use-amazon-route53-domain-api has a great guide on how to do this. Be sure to checkout this document as well. https://github.com/acmesh-official/acme.sh/wiki/How-to-use-Amazon-Route53-API

```bash
# Install Letsencrypt
sudo su -
curl https://get.acme.sh | sh
/root/.acme.sh/acme.sh --set-default-ca --server letsencrypt
```

## NGINX Configuration

Before we get into the thick of it, lets get nginx installed.

```bash
sudo su -
apt-get update

# Install nginx and php deps
apt-get install nginx subversion ssl-cert php-fpm libfcgi0ldbl php-cli php-dev php-xml php-curl php-xmlrpc php-json php-mbstring php-opcache php-xml php-zip -y

# Configure PHP
cd /etc/php
phpv=$(ls -d */ | cut -d/ -f1)
echo_progress_start "Making adjustments to PHP"
for version in $phpv; do
    sed -i -e "s/post_max_size = 8M/post_max_size = 64M/" \
        -e "s/upload_max_filesize = 2M/upload_max_filesize = 92M/" \
        -e "s/expose_php = On/expose_php = Off/" \
        -e "s/128M/768M/" \
        -e "s/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/" \
        -e "s/;opcache.enable=0/opcache.enable=1/" \
        -e "s/;opcache.memory_consumption=64/opcache.memory_consumption=128/" \
        -e "s/;opcache.max_accelerated_files=2000/opcache.max_accelerated_files=4000/" \
        -e "s/;opcache.revalidate_freq=2/opcache.revalidate_freq=240/" /etc/php/$version/fpm/php.ini
    phpenmod -v $version opcache
done

# Generate filestructure and dhparam
mkdir -p /etc/nginx/ssl
mkdir -p /etc/nginx/snippets
cd /etc/nginx/ssl
openssl dhparam -out dhparam.pem 2048

mkdir -p /srv

# Install librespeed
git clone "https://github.com/librespeed/speedtest.git" "/srv/librespeed"
cp /srv/librespeed/example-singleServer-gauges.html /srv/librespeed/index.html
chown www-data: /srv -R
```

In this repo, you'll find a configuration file called `magic.conf`. This deals with the proxying of `srv1.example.com` to `srv1.cdn.example.com`. Essentially, we're going to take the value of `$host` in the request, and map it to the `$og_host`. This means that, in essence you will be setting the first value to `srv1.cdn.example.com` and the second value to `srv1.example.com`. You need to either create 2 configs on the host? 9or be able to use wildcard certs to allow it to serve both on the normal and CDN configs  to create a normal config, copy the cdn config, remove the block for images, then change all instances of `og_host` to `host`.

Because `plex-cdn.conf` `include snippets/magic.conf`, it's going to proxy the request, and replace `$og_host` in all instances with `srv1.example.com`. This configuration is SNI compliant, and will forward the appropriate SSL details from the host, allowing you to use a secondary host, like Traefik on a Saltbox instance to host the Plex. We also include a speedtest (via librespeed) for our users so they can speedtest and we can gether diagnostic data if something is wrong.

The advantage of using this also extends to caching the images themselves (IE the posters), via the block found at line 69-85 of `sites-enabled/plex-cdn.conf`.

Basically, you'll need to dump the contents of this repository into `/etc/nginx/` and pray.

If you're already running swizzin, this should be fairly simple to get going side-by-side.

## UML Diagram

![CDN Configuration](https://raw.githubusercontent.com/brettpetch/plex-cdn/main/cdn.png)

## Special Thanks

Rox, for the amazing little ditty that makes plex cache images on the edge

The [Swizzin Project](https://github.com/swizzin/swizzin) for NGINX install, PHP configs

You for making this possible.
