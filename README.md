# Self-Hosted Plex CDN

So... you alike me, brave traveler have ended up here in search of a way to make your Plex Media Server instance peer better between you and your host. Worry not, we have been working on this as our pet project for months and will give you the crash course on how this fucking thing works.

## Requirements

- An AWS account (with billing and a hosted zone).
- A few servers laying around (ideally 1gbps+).
- A number of beverages (preferably beers, find a nice IPA...)

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

Basically we're going to proxy and cache (images) as they're requested. Using some of the persistent storage at the edge. You'll need to modify your nginx config (see the one that's in here for what to change).

I personally host my Plex offshore. The price of storage is so low that tossing a US bouncer on is far cheaper than hosting it in the US entirely. Especially with the peering issues that places like WebNX, and in my experience, OVH have had connecting directly to storage.

By dynamically routing traffic in this manner you are able to ignore ISP prioritization by masking video as normal traffic and cache images at the edge for a better experience.

While there are ways to commercialize this, I don’t have a reason to really. It’s simply something that I wanted to do to speed up page loads and bounce a few instances for friends (who use traefik, nginx, and caddy) through the setup.

## UML Diagram

![CDN Configuration](https://raw.githubusercontent.com/brettpetch/plex-cdn/main/cdn.png)

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

### SWAG Setup

SWAG introduces a number of new issues, one of which being that you can't do unlimited depth in subdomaining easily.

As an alternative solution, you can instead create a new config (copy plex.subdomain.conf.example) and create a second file called plex-cdn.subdomain.conf. 

Inside this file, change the server_name to plex-cdn.*

Then leave everything else the same. On AWS Route53, you'll also need to change the geodns records to plex-cdn. ENSURE THAT YOU HAVE A DEFAULT RECORD FOR plex-cdn! Otherwise it can break domain name resolution in other countries. 

### Final Notes

In Plex, you need to change the advanced network settings to use the custom host `https://srv1.cdn.example.com:443,http://srv1.cdn.example.com:80` if you want it to use the dynamic routing. This is the url that is reported to clients. You'll also need remote access disabled, and an nginx config for plex itself (not a cdn config)

## FAQ

> What kind of cost does this incur?

As already mentioned, I run 2 domains and it costs about $4.20/mo for GeoDNS resolution. On top of that, you've got however many droplets / VPSes that you decide to use. I'd recommend at least one in each continent.

> I’m assuming it doesn’t cache things like video previews right? 

This will cache any transcoded images on the edge for the duration of its TTL, which Plex sets upstream. It will respect this.

> How many servers do I need?

You'll need at a minimum a 1 Plex server, 1 instance of nginx running beside that Plex server, and 1 external server (IE a VPS) to effectively route traffic the way that this guide describes. If you have the ability, I'd try for a VPS in each region, after testing throughput to your origin and to your clients.

> But wait... Couldn't I just use a 302 to the correct server for the region and not pay for R53?

From my testing, this didn't work. It failed spectacularly. If you can get this to work, do give me a shout. I could be wrong, but my experience was that Plex client applications would not respond to 302s with rewrites from GeoIP DB mappings.

## Special Thanks

[Rox](https://github.com/Roxedus), for the amazing little ditty that makes plex cache images on the edge

The [Swizzin Project](https://github.com/swizzin/swizzin) for NGINX install, PHP configs

You for making this possible.

## About me

I am a recent grad from Western University, where I studied Media, Information, and Technoculture; or as you might know it "Media Studies." Now I've graduated with crippling amounts of student debt, and instead of paying it all off... I ended up building a CDN and writing this guide that you're reading now... If this was a help to you, please consider [checking out my GH Sponsors Page](https://github.com/sponsors/brettpetch).
