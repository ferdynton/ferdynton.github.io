---
title: "Migrating a Rails app from Heroku to a Raspbery Pi"
date: 2019-12-22T20:53:02-06:00
draft: false
---

I've got a Rails app called [MovieMates](https://moviemates.party) that I've
been building on my spare time for a few years now. When it got mature enough, I
deployed it to Heroku for 3 reasons:

* Convenience (a basic deployment for Rails is as simple as `git push heroku`,
  and a lot of the tools out there are optimized for Heroku, including Rails
  itself)
* [Free PostgreSQL
  database](https://elements.heroku.com/addons/heroku-postgresql) (which for a
  toy app like mine, means less money and peace of mind due to auto-management
  and free backups)
* [Automatic TLS certificate
  management](https://devcenter.heroku.com/articles/automated-certificate-management)

However, there is one disadvantage:

* [A Hobby dyno is $84 a year](https://www.heroku.com/pricing) (this feels like
  a lot of money for an app very few people use and makes zero money. I also
  happen to be a very frugal person)

I recently decided I wanted to start saving those $84 a year and I migrated my
deployment to a Raspberry Pi I've got running at home. This wasn't easy, but it
was fun and I learned a lot.

I want to share all the steps I took in detail in case it helps anybody out
there (or myself in the future, if I have to do this process again!).

Here is a summary of the steps to take from the Raspberry Pi (assuming it's
running a Raspbian-like OS and you have a home network with a router that
supports NAT):

* [Create a dedicated user with SSH access](#create-user)
* [Install Nginx, chruby, PostgreSQL, and NVM](#install)
* [Migrate environment variables](#env-vars)
* [Run the app from the Raspberry Pi](#run-app)
* [Configure the `mina` gem](#mina)
* [Daemonize the Rails app with `systemd` and Puma](#daemonize)
* [Configure Nginx](#nginx)
* [Configure your router and domain](#domain)
* [Install a TLS certificate with LetsEncrypt](#letsencrypt)

And a couple of bonus sections:

* [Auto-deploy from GitLab](#auto-deploy)
* [Auto-backup the database regularly](#auto-backup)

## Create a dedicated user with SSH access {#create-user}

For security and organizational purposes, it's good practice to keep this whole
adventure managed under a new and separate user on the Raspberry Pi. We'll call
it `rails` in this tutorial, but you might want to call it whatever your app is
called.

```sh
sudo useradd -m -s zsh rails
```

I've set the shell to Zsh because I like it better than Bash. The rest of the
tutorial will use dotfiles that start like `.zsh...` instead of `.bash...`, but
otherwise the two shells should be pretty much equivalent.

Now you want to be able to SSH into your Raspberry Pi as this user because
that will be useful for using the deployment technique. To do this, first open a
shell with that user through any other user that has `sudo` access:

```sh
sudo su - rails
```

Then, create an `.ssh/authorized_keys` file with the contents of your public key
from the machine you're using to develop:

```sh
mkdir .ssh
echo 'ssh-rsa AAAAB3Nza...XYZ development-machine' > .ssh/authorized_keys
```

If you don't know where your public key is in your system, look for it in
`~/.ssh/id_rsa.pub`. If such file doesn't exist, you can create your first pair
of private/public keys using the `ssh-keygen` command and accepting all the
defaults.

## Install Nginx, `chruby`, PostgreSQL, and NVM {#install}

All of these pieces of software might be optional depending on your preferences,
but I went for this setup. Nginx has been widely used for many years as a
reverse proxy. `chruby` is the simplest way of managing Ruby versions and we
won't need many features from a Ruby version manager throughout this tutorial.
PostgreSQL is the most popular choice of database for Rails applications, but
your app might be using another one. Finally, if you rely on Sprockets for the
asset pipeline, you'll need Node, for which I recommend NVM (Node Version
Manager) so we can upgrade easily in the future if we need (rather than using
the system's Node installation).

### Nginx

```sh
sudo apt install nginx
```

### `chruby`

Refer to [the repo's installation
instructions](https://github.com/postmodern/chruby#install), which are probably
something close to:

```sh
wget -O chruby-0.3.9.tar.gz https://github.com/postmodern/chruby/archive/v0.3.9.tar.gz
tar -xzvf chruby-0.3.9.tar.gz
cd chruby-0.3.9/
sudo make install
```

You'll probably want `ruby-install`, so use [the repo's installation
instructions](https://github.com/postmodern/ruby-install#install), which are
probably close to:

```sh
wget -O ruby-install-0.7.0.tar.gz https://github.com/postmodern/ruby-install/archive/v0.7.0.tar.gz
tar -xzvf ruby-install-0.7.0.tar.gz
cd ruby-install-0.7.0/
sudo make install
```

Install the Ruby version for your app using `sudo`, which will make it available
to all users in the system:

```sh
sudo ruby-install ruby-2.5.7
```

Finally load it from the `.zshenv` file in the `$HOME` directory for your
`rails` user, including your desired Ruby version:

```sh
source /usr/local/share/chruby/chruby.sh
chruby 2.5.7
```

More about the `.zshenv` file on [the environment variables section](#env-vars).

### PostgreSQL

Installing PostgreSQL is easy:

```sh
sudo apt install postgresql
```

Once that's done, you'll want to create a database, a database user for your
`rails` user account, and migrate the data from your existing database. To
perform these steps, you'll want to use the `postgres` user that should have now
been created in the Raspberry Pi:

```sh
sudo su - postgres
```

From here, create your production database (use whatever database name your
current app is using):

```sh
createdb your_app_database_prod
```

Now create a user with a password. Any user name would do, but I think it's
consistent to reuse the name you used for your user account, in this case
`rails`. To generate a strong random password, you can run `bundle exec rails
secret` from your development machine to get a bunch of random characters (no
need to get the full string, 30 or 40 characters should be enough). Log into a
PostgreSQL console to launch the appropriate command:

```sh
psql your_app_database_prod

your_app_database_prod=# CREATE ROLE rails WITH LOGIN PASSWORD '<long-random-string>'
```

Save the password for later! You can exit the PostgreSQL console now.

You might want to import your database from Heroku. One way of getting a copy of
the current database is to go to [Heroku's Data
Center](https://data.heroku.com/) and selecting the database for your app. From
there, go to "Durability" > "Create Manual Backup". Give it a few seconds to
complete, then "Download" it.

While on a shell for the `postgres` user, run `pg_restore` with the parameters
that make sense for your app.

```sh
pg_restore -d your_app_database_prod /path/to/backup/file
```

### NVM

Like `chruby`, make sure to refer to [the repo's installation
instructions](https://github.com/nvm-sh/nvm#installation-and-update). Unlike
`chruby`, NVM is not prepared to be installed system-wide, so **make sure you
install this from your `rails` user account**. It should be something like:

```sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.2/install.sh | bash
```

Then install the latest Node version available:

```sh
nvm ls-remote       # List all available versions
nvm install v13.2.0 # Install the latest you see
```

Finally, add these lines to your `.zshenv` file:

```sh
export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"
```

More about the `.zshenv` file on [the environment variables section](#env-vars).

## Migrate environment variables {#env-vars}

To migrate environment variables, `export` them in the `.zshenv` file in the
`$HOME` directory for your `rails` user:

```sh
# Rails app environment variables

export DISABLE_BOOTSNAP=1
export RAILS_ENV=production
export RACK_ENV=production

# ... and any others you've got!
```

Per [ZSH's documentation](http://zsh.sourceforge.net/Intro/intro_3.html), the
`.zshenv` file is sourced on all invocations of the shell, including
non-interactive ones. This is important because [our deployment script](#mina)
won't use an interactive shell, but loging into the user account from a SSH
session will work perfectly as well.

Important mention to the `DISABLE_BOOTSNAP` environment variable I included in
the example above: at the time of writing, there's [a bug in Bootsnap with Ruby
2.5 and below](https://github.com/Shopify/bootsnap/issues/67) that prevents
Rails to boot on an architecture like the one a Raspberry Pi has. I have my
Rails application patched to not use Bootsnap in production:

```ruby
# config/boot.rb

# ...

require "bootsnap/setup" unless ENV["DISABLE_BOOTSNAP"]
```

## Run the app from the Raspberry Pi {#run-app}

At this point, you should be able to run your app from your `rails` user on your
Raspberry Pi! This step is not technically necessary, but I think it's a good
sanity check to make sure things are working fine up to this point.

First, clone your repository:

```sh
git clone https://gitlab.com/Ferdy89/movie_mates.git
cd movie_mates
```

Then, install all the dependencies:

```sh
bundle install
```

Finally, serve your app!

```sh
bundle exec rails server
```

If everything went right, you should be able to `curl` your `localhost` on
whatever port your app is serving (usually 3000 by default) and see the HTML
code of your home page:

```sh
curl -L localhost:3000

<!doctype html>
<html>

# ... more HTML code
```

If so, congrats! If not, make sure to go back and make everything right up to
this point, because harder stuff is coming!

## Configure the `mina` gem {#mina}

[`mina`](https://github.com/mina-deploy/mina) is a gem to deploy applications
over SSH. It's like `capistrano`, but faster.

This step should be performed on the machine where you develop your app, not
from Raspberry Pi. First, add it to your `Gemfile` or to a separate `Gemfile`
you might want to keep for gems you don't use within your application directly.
I do this with my apps to keep things separate. I call it `Gemfile-tools`, but
any name works:

```ruby
# Gemfile-tools

# Activate with BUNDLE_GEMFILE=Gemfile-tools

gem 'mina'
```

And install it:

```sh
BUNDLE_GEMFILE=Gemfile-tools bundle install
```

Now, following [`mina`'s
guide](https://github.com/mina-deploy/mina/blob/master/docs/getting_started.md),
initialize the files required for the gem:

```sh
BUNDLE_GEMFILE=Gemfile-tools mina init
```

This should create a `config/deploy.rb` file. Here's how you want this file to
look for the time being:

```ruby
require "mina/rails"
require "mina/git"

set :application_name, "your_app_name"

# The hostname to SSH to.
set :domain, "your.raspberry.pi.domain"

# Username in the server to SSH to.
set :user, "rails"

# Path to deploy into.
set :deploy_to, "/var/www/your_app_name"

# Git repo to clone from. (needed by mina/git)
set :repository, "https://github.com/YourUser/your_app_name.git"

# Branch name to deploy. (needed by mina/git)
set :branch, "master"

desc "Deploys the current version to the server."
task :deploy do
  deploy do
    invoke :'git:clone'
    invoke :'deploy:link_shared_paths'
    invoke :'bundle:install'
    invoke :'rails:db_migrate'
    invoke :'rails:assets_precompile'
    invoke :'deploy:cleanup'
  end
end
```

You'll need to replace the `set` commands at the top of the file with whatever
values are currently true for your app.

Don't commit this file yet! You might want to make the script pull some of those
values from your environment variables to avoid leaking unnecessary information
to the world. I'll teach you how to do that in the [auto-deploy
section](#auto-deploy).

You can now proceed to have `mina` get installed on _the other side_, meaning
your Raspberry Pi.

```sh
BUNDLE_GEMFILE=Gemfile-tools mina setup
```

If this command succeeds, it means `mina` is perfectly capable of taking to your
Raspberry Pi, which is huge!

At this point, the `mina` guide tells us to run `mina deploy`. We're going to do
that so that there's a copy of the app in the Raspberry Pi the same way it'll be
in production.

```sh
BUNDLE_GEMFILE=Gemfile-tools mina deploy
```

## Daemonize the Rails app with `systemd` and Puma {#daemonize}

We're now in position to run the app from the Raspberry Pi and have `systemd` run
it forever, even if the system restarts or if the app crashes.

Puma has great docs on [how to configure it with
`systemd`](https://github.com/puma/puma/blob/master/docs/systemd.md). In
essence, those instructions teach you how to install a service to keep the app
running as the system boots and to survive crashes, and also how to keep an open
socket so the app doesn't requests don't automatically fail while the app is
being deployed or rebooted.

For the purposes of this tutorial, I created my own [file to install the
service](https://gitlab.com/Ferdy89/movie_mates/blob/master/deploy/puma.service)
and my own [file to install the
socket](https://gitlab.com/Ferdy89/movie_mates/blob/master/deploy/puma.socket).
Both come with instructions on how to use them. You want to copy them to
`/etc/systemd/system/` and replace `<DEPLOY_USER>` with `rails`, and
`<DEPLOYMENT_PATH>` with `/var/www/your_app_name` (as defined in your `mina`
config file). After that, run the commands:

```sh
systemctl daemon-reload                   # Picks up the new files
systemctl enable puma.socket puma.service # Installs the socket and the service
systemctl start puma.socket puma.service  # Starts the socket and the service
```

In order for `mina` to be able to restart this service whenever a deployment
happens, we need to allow the `rails` user to perform this restart, which is a
fairly privileged action. From your Raspberry Pi, create a `sudoers` file for
your `rails` user with a rule to allow to run one single command without a
password:

```sh
sudo echo "rails ALL= NOPASSWD: /bin/systemctl restart puma.service" > /etc/sudoers.d/rails
```

Then add an action to the end of your `config/deploy.rb` file so `mina` can
restart the Puma service after it has deployed the new code:

```ruby
# config/deploy.rb

# ...

task :deploy do
  deploy do
    # ...

    on :launch do
      command "sudo systemctl restart puma.service"
    end
  end
end
```

One last note about the `puma.service` file: it requires your app to have
binstubs for Puma. This can be created on your app by running:

```sh
bundle binstubs puma
```

Then commit the changes to your repository and run `mina deploy` again. This
makes it easier to call Puma on the repository and have it use the right
Bundler installation.

## Configure Nginx {#nginx}

At this point, `systemd` is keeping the app running but it's listening on a
socket and we want to be able to access it from the outside world through HTTP.
That's what Nginx is for.

Once again, I've made my own version of the [Nginx configuration
file](https://gitlab.com/Ferdy89/movie_mates/blob/master/deploy/nginx-movie_mates),
which you can tweak to fit your own needs. You only need to copy it to
`/etc/nginx/sites-available/` and symlink to `/etc/nginx/sites-enabled/`. It's
specific to the `moviemates.party` domain, so replace that with whatever domain
you have for your app. Finally, replace `<DEPLOYMENT_PATH>` with
`/var/www/your_app_name` like in previous steps.

There are some entries in there that are specific to LetsEncrypt and the way it
manages TLS certificates. We won't be getting these certificates until a couple
of steps later, so trust me for now.

## Configure your router and domain {#domain}

This part is a bit trickier, because it'll depend on what router you have and
what kind of firmware you have installed. I've got a [$20 TP-LINK
router](https://www.amazon.com/TP-LINK-300Mbps-Wireless-Router-TL-WR841N/dp/B01GS37NV6/)
that I got many years ago and still works fine. I've got the [DD-WRT open
firmware distribution](https://dd-wrt.com/) installed and among the million
features it has, it can route traffic from the WAN network (the Internet) to a
local machine and port (your Raspberry Pi). I've had issues with this in the
past, but I [recently posted on their
forums](https://forum.dd-wrt.com/phpBB2/viewtopic.php?p=1184428) and they helped
me fix my problem (spoiler alert: I needed to install a nightly version).

If you're using DD-WRT, go to [your control panel](http://192.168.1.1), then to
"NAT/Qos" and, while in the "Port Forwarding" tab, add an entry for your
application. Use whatever name you want under "Application", "Protocol" is
"TCP", "Source Net" is "0.0.0.0/0" (the Internet), "Port from" is 80, "IP
Address" is the local address of your Raspberry Pi (you can find it by running
`ifconfig` on your Raspberry Pi and finding the `inet` parameter that starts
with `192.168`), and "Port to" is 8080. Check "Enable", then "Apply Settings".

This should now have your router open up traffic from the Internet to your
Raspberry Pi. Scary, uh? This is a big trade-off of this self-hosting approach:
if you make a mistake or become a target for whatever reason, an attacker might
be able to get into your home network and do bad things from there. You need to
consider whether this is a good strategy for you. The configuration I've shared
here is the one I personally use and I feel comfortable with it, but if you've
got important stuff going on in your home network or if you're targeted by
anybody on the Internet, I'd suggest reconsidering opening up any ports out to
the Internet.

Finally, have whatever domain you have for your application to point to your
home network. Do this by logging into your DNS management portal for your domain
provider (I use Namecheap because, well, they're cheap) and creating an "A
Record" to "Host" "@" with a "Value" of your Internet IP (find out about it by
Googling "what is my ip"). Give it a few minutes to propagate, otherwise the
next step might fail.

## Install a TLS certificate with LetsEncrypt {#letsencrypt}

The last step before getting your site up and running like a professional one is
to get a TLS certificate to have Nginx encrypt all communications and browsers
knowing that they can trust these communications.

This is mostly automated, here are the steps you need to take:

1. Install `certbot` on your Raspberry Pi by following [the instructions on
   their website](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx).
2. Temporarily route to port 80 on your machine. Go to the same "Port
   Forwarding" tab on your router as in the last step and change "Port to" to be
   80, then "Apply Settings". We'll revert this afterwards.
3. Run `certbot` to verify your domain and install your certificate by running
   `sudo certbot --nginx -d moviemates.party` (use your own domain here).
4. Once `certbot` succeeds, go back to your "Port Forwarding" tab on your router
   control panel and put "Port to" back to 8080 and "Apply Settings".
5. Finally, open your Nginx configuration file on `/etc/nginx/sites-available/`
   and search for any new changes that LetsEncrypt might have added. Delete any
   duplicate lines to avoid conflicts.

Yay! You did it! If everything went right, you should now be able to hit your
application domain from the Internet (I prefer doing this from my phone without
using WiFi to ensure the networking configuration is correct) and see your home
page! And with HTTPS! If this is not the case, please leave me a comment and let
me know so I can help you and improve this guide for others.

## Auto-deploy from GitLab {#auto-deploy}

At this point, your app is up and running and you can always manually deploy any
changes using `mina deploy` from your laptop. However, I prefer to follow the
principles of Continuous Deployment and avoid having to perform that extra step
every time. This step teaches you how to get your GitLab-hosted project to be
automatically deployed every time a merge to the `master` branch happens.

First, add the following step to your `.gitlab-ci.yml` file:

```yaml
production:
  stage: deploy
  cache: {}
  script:
    # https://docs.gitlab.com/ee/ci/ssh_keys/#ssh-keys-when-using-the-docker-executor
    - eval $(ssh-agent -s)
    - echo "$MINA_SSH_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh

    # https://docs.gitlab.com/ee/ci/ssh_keys/#verifying-the-ssh-host-keys
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts

    - BUNDLE_GEMFILE=Gemfile-tools bundle exec mina deploy
  only:
    - master
```

Then create a pair of public/private keys so GitLab can securely talk to your
Raspberry Pi. Do this by running `ssh-keygen` from any machine and save it with
the name `gitlab-key`. Once you've created them, grab the public one (file
`gitlab-key.pub`) and copy its contents to the `~/.ssh/authorized_keys` file on
your `rails` user account on your Raspberry Pi.

After this, on your GitLab project go to "Settings", "CI/CD", "Variables". This
is where you can host private environment variables for your CI/CD pipeline to
use without having to commit them to code. Make sure to mark them all as
"Protected" so their values are scrubbed from the CI/CD output. For now, create
a `MINA_SSH_KEY` variable and paste the contents of your `gitlab-key` private
key (file `gitlab-key`). Once you've done this, you can delete both the
`gitlab-key` and `gitlab-key.pub` files from whatever machine you used to
create them.

Unfortunately, your router still won't let GitLab talk to your Raspberry Pi. Go
back to your "Port Forwarding" tab on your router's control panel and add an
entry for this. Give it a name, protocol TCP, source net "0.0.0.0/0" (the
Internet), choose a random number for the "Port from", then use the local IP
address of your Raspberry Pi and set "Port to" to 22. Then "Apply Settings".
Before you forget, create another environment variable on your GitLab project
and name it `MINA_PORT`, then use the value of the random port you used on your
router entry.

To add another layer of safety in the connection and avoid man-in-the-middle
attacks, let's store the fingerprint of your Raspberry Pi on GitLab. From your
laptop, run:

```
ssh-keyscan -p <your-random-port-number> <your-public-domain>`
```

Save the output of this command in another environment variable on your GitLab
project, named `SSH_KNOWN_HOSTS`.

Finally, as discussed in the step about [how to configure `mina`](#mina), we can
now hide the rest of the parameters of the connection to our Raspberry Pi so we
don't give that information for free to the public. Create these environment
variables with these values:

```
MINA_DEPLOYMENT_PATH </var/www/your_app_name>
MINA_DOMAIN <your-public-domain>
MINA_USER <rails in this tutorial>
```

Now you can change your `config/deploy.rb` file to use these parameters:

```
# config/deploy.rb

# The hostname to SSH to.
set :domain, ENV.fetch("MINA_DOMAIN")

# Username in the server to SSH to.
set :user, ENV.fetch("MINA_USER")

# SSH port number.
set :port, ENV.fetch("MINA_PORT")

# Path to deploy into.
set :deploy_to, ENV.fetch("MINA_DEPLOYMENT_PATH")
```

If everything went right, every Merge Request that gets merged to `master`
should now trigger a deployment from GitLab to your Raspberry Pi!

## Auto-backup the database regularly {#auto-backup}

If you're anything like me, you're very paranoid of these Raspberry Pis. I love
them because they're cheap and sufficiently powerful but they're not as stable
as Heroku, which means it'll eventually crash and die. This will mean you'll
need to reinstall most of the things from this guide. This is painful, but will
probably take you about an hour or two, no big deal. The worst loss is your
database: whatever your app was storing in your Raspberry Pi is lost forever.

That's not acceptable. I've got mine set up so it backs up the whole database to
an external hard drive every hour, so I'm pretty sure I'm not going to lose much
data even if the worst happens.

To do this, log as your `postgres` user on your Raspberry Pi (it has access to
all databases) and insert a new cron job:

```sh
sudo su - postgres
crontab -e

# Add the following line
0 * * * * pg_dump -f /<path-to-your-external-hard-drive>/your_app_database_prod-`date +\%Y-\%m-\%d_\%H-\%M-\%S` -Z 9 your_app_database_prod
```

This will create a new compressed backup every hour with a timestamp. If you
ever have a catastrophe, you can use the command `pg_restore` to get any of
these files and restore the state of your app the way it was.
