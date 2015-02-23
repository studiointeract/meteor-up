# Meteor Up

#### Production Quality Meteor Deployments

Meteor Up (mup for short) is a command line tool that allows you to deploy any [Meteor](http://meteor.com) app to your own server. It supports only Debian/Ubuntu flavours and Open Solaris at the moments. (PRs are welcome)

> Screencast: [How to deploy a Meteor app with Meteor Up (by Sacha Greif)](https://www.youtube.com/watch?v=WLGdXtZMmiI)

**Table of Contents**

- [Features](#features)
- [Server Configuration](#server-configuration)
- [Nginx Multiple App Hosting](#server-nginx)
- [Installation](#installation)
- [Creating a Meteor Up Project](#creating-a-meteor-up-project)
- [Example File](#example-file)
- [Setting Up a Server](#setting-up-a-server)
    - [Server Setup Details](#server-setup-details)
- [Deploying an App](#deploying-an-app)
    - [Deploy Wait Time](#deploy-wait-time)
    - [Multiple Deployment Targets](#multiple-deployment-targets)
- [Access Logs](#access-logs)
- [Reconfiguring & Restarting](#reconfiguring--restarting)
- [Accessing the Database](#accessing-the-database)
- [Multiple Deployments](#multiple-deployments)
- [Server Specific Environment Variables](#server-specific-environment-variables)
- [SSL Support](#ssl-support)
- [Updating](#updating)
- [Troubleshooting](#troubleshooting)
- [Binary Npm Module Support](#binary-npm-module-support)
- [Additional Resources](#additional-resources)

### Features

* Single command server setup
* Single command deployment
* Multi server deployment
* Environmental Variables management
* Support for [`settings.json`](http://docs.meteor.com/#meteor_settings)
* Password or Private Key(pem) based server authentication
* Access, logs from the terminal (supports log tailing)
* Support for multiple meteor deployments (experimental)

### Server Configuration

* Auto-Restart if the app crashed (using forever)
* Auto-Start after the server reboot (using upstart)
* Stepdown User Privileges
* Revert to the previous version, if the deployment failed
* Secured MongoDB Installation (Optional)
* Pre-Installed PhantomJS (Optional)

### Server Nginx

* Host multiple applications on a single server
* Define a unique port and ROOT_URL for each instance and Nginx will proxy forward accordingly
* Prevents multiple deployments of the same port on a single server
* add `setupNginx:true` to `mup.json` and specify a `PORT` and `ROOT_URL` in the environment variables
* Provide a unique `appName` for each app. This will install it into its own folder and own upstart on the server.
* Serve files from the `/public` folder directly through Nginx with caching instead of Meteor directly `(jpg|jpeg|png|gif|mp3|ico|pdf)`

### Installation

    npm install -g mup

If you are looking for password-based authentication, you need to [install sshpass](https://gist.github.com/arunoda/7790979) on your local development machine.

### Creating a Meteor Up Project

    mkdir ~/my-meteor-deployment
    cd ~/my-meteor-deployment
    mup init

This will create two files in your Meteor Up project directory:

  * mup.json - Meteor Up configuration file
  * settings.json - Settings for Meteor's [settings API](http://docs.meteor.com/#meteor_settings)

`mup.json` is commented and easy to follow (it supports JavaScript comments).

### Example File

```js
{
  // Server authentication info
  "servers": [
    {
      "host": "hostname",
      "username": "root",
      "password": "password"
      // or pem file (ssh based authentication)
      // WARNING: Keys protected by a passphrase are not supported
      //"pem": "~/.ssh/id_rsa"
      // Also, for non-standard ssh port use this
      //"sshOptions": { "Port" : 49154 },
      // server specific environment variables
      "env": {}
    }
  ],

  // Install MongoDB on the server. Does not destroy the local MongoDB on future setups
  "setupMongo": true,

  // WARNING: Node.js is required! Only skip if you already have Node.js installed on server.
  "setupNode": true,

  // WARNING: nodeVersion defaults to 0.10.33 if omitted. Do not use v, just the version number.
  "nodeVersion": "0.10.33",

  // Install PhantomJS on the server
  "setupPhantom": true,

  // Install Nginx on the server (Requires a unique PORT and ROOT_URL in the environment settings
  // Note: Port must not be 80 as Nginx will proxy_forward to port 80 per ROOT_URL (e.g. 3000, 3001, 3002, etc)
  "setupNginx": true,

  // Application name (no spaces).
  "appName": "meteor",

  // Location of app (local directory). This can reference '~' as the users home directory.
  // i.e., "app": "~/Meteor/my-app",
  // This is the same as the line below.
  "app": "/Users/arunoda/Meteor/my-app",

  // Configure environment
  // ROOT_URL must be set to https://YOURDOMAIN.com when using the spiderable package & force SSL
  // your NGINX proxy or Cloudflare. When using just Meteor on SSL without spiderable this is not necessary
  "env": {
    "PORT": 80,
    "ROOT_URL": "http://myapp.com",
    "MONGO_URL": "mongodb://arunoda:fd8dsjsfh7@hanso.mongohq.com:10023/MyApp",
    "MAIL_URL": "smtp://postmaster%40myapp.mailgun.org:adj87sjhd7s@smtp.mailgun.org:587/"
  },

  // Meteor Up checks if the app comes online just after the deployment.
  // Before mup checks that, it will wait for the number of seconds configured below.
  "deployCheckWaitTime": 15
}
```

### Setting Up a Server

    mup setup

This will setup the server for the `mup` deployments. It will take around 2-5 minutes depending on the server's performance and network availability.

#### Ssh based authentication

Please ensure your key file (pem) is not protected by a passphrase. Also the setup process will require NOPASSWD access to sudo. (Since Meteor needs port 80, sudo access is required.)

You can add your user to the sudo group:

    sudo adduser *username*  sudo

And you also need to add NOPASSWD to the sudoers file:

    sudo visudo

    # replace this line
    %sudo  ALL=(ALL) ALL

    # by this line
    %sudo ALL=(ALL) NOPASSWD:ALL  

When this process is not working you might encounter the following error:

    'sudo: no tty present and no askpass program specified'

#### Server Setup Details

This is how Meteor Up will configure the server for you based on the given `appName` or using "meteor" as default appName. This information will help you customize the server for your needs.

* your app lives at `/opt/<appName>/app`
* mup uses `upstart` with a config file at `/etc/init/<appName>.conf`
* you can start and stop the app with upstart: `start <appName>` and `stop <appName>`
* logs are located at: `/var/log/upstart/<appName>.log`
* MongoDB installed and bound to the local interface (cannot access from the outside)
* the database is named `<appName>`

For more information see [`lib/taskLists.js`](https://github.com/arunoda/meteor-up/blob/master/lib/taskLists.js).

### Deploying an App

    mup deploy

This will bundle the Meteor project and deploy it to the server.

#### Deploy Wait Time

Meteor Up checks if the deployment is successful or not just after the deployment. By default, it will wait 10 seconds before the check. You can configure the wait time with the `deployCheckWaitTime` option in the `mup.json`.

#### Multiple Deployment Targets

You can use an array to deploy to multiple servers at once.

To deploy to *different* environments (e.g. staging, production, etc.), use separate Meteor Up configurations in separate directories, with each directory containing separate `mup.json` and `settings.json` files, and the `mup.json` files' `app` field pointing back to your app's local directory.

#### Custom Meteor Binary

Sometimes, you might be using `mrt`, or Meteor from a git checkout. By default, Meteor Up uses `meteor`. You can ask Meteor Up to use the correct binary with the `meteorBinary` option.

~~~js
{
  ...
  "meteorBinary": "~/bin/meteor/meteor"
  ...
}
~~~

### Access Logs

    mup logs -f

Mup can tail logs from the server and supports all the options of `tail`.

### Reconfiguring & Restarting

After you've edit environmental variables or `settings.json`, you can reconfigure the app without deploying again. Use the following command to do update the settings and restart the app.

    mup reconfig

If you want to stop, start or restart your app for any reason, you can use the following commands to manage it.

    mup stop
    mup start
    mup restart

### Accessing the Database

You can't access the MongoDB from the outside the server. To access the MongoDB shell you need to log into your server via SSH first and then run the following command:

    mongo appName

### Server Specific Environment Variables

It is possible to provide server specific environment variables. Add the `env` object along with the server details in the `mup.json`. Here's an example:

~~~js
{
  "servers": [
    {
      "host": "hostname",
      "username": "root",
      "password": "password"
      "env": {
        "SOME_ENV": "the-value"
      }
    }

  ...
}
~~~

By default, Meteor UP adds `CLUSTER_ENDPOINT_URL` to make [cluster](https://github.com/meteorhacks/cluster) deployment simple. But you can override it by defining it yourself.

### Multiple Deployments

Meteor Up supports multiple deployments to a single server. Meteor Up only does the deployment; if you need to configure subdomains, you need to manually setup a reverse proxy yourself.

Let's assume, we need to deploy production and staging versions of the app to the same server. The production app runs on port 80 and the staging app runs on port 8000.

We need to have two separate Meteor Up projects. For that, create two directories and initialize Meteor Up and add the necessary configurations.

In the staging `mup.json`, add a field called `appName` with the value `staging`. You can add any name you prefer instead of `staging`. Since we are running our staging app on port 8000, add an environment variable called `PORT` with the value 8000.

Now setup both projects and deploy as you need.

### SSL Support

Meteor Up now supports both stud and nginx SSL termination for your app.

#### Nginx SSL Configuration

* Place your certficate and private key files in a folder (suggestion is to use `.ssl` in your project folder)
* Add the following lines to your `mup.json` file:

~~~js
{
  ...

  "ssl": {
    "pem": "./.ssl/cert.pem",
    "key": "./.ssl/private.key"
  }

  ...
}
~~~

* Run `mup deploy` on a preconfigured Nginx+MUP host.
* you will now be able to browse to your website address and it will automatcially promote it to SSL.

**Note:** Be sure to add the `https://` path to your `ROOT_URL` enviromnent variable also!

#### Stud SSL Configuration

Meteor Up has the built in SSL support. It uses [stud](https://github.com/bumptech/stud) SSL terminator for that. First you need to get a SSL certificate from some provider. This is how to do that:

* [First you need to generate a CSR file and the private key](http://www.rackspace.com/knowledge_center/article/generate-a-csr-with-openssl)
* Then purchase a SSL certificate.
* Then generate a SSL certificate from your SSL providers UI.
* Then that'll ask to provide the CSR file. Upload the CSR file we've generated.
* When asked to select your SSL server type, select it as nginx.
* Then you'll get a set of files (your domain certificate and CA files).

Now you need combine SSL certificate(s) with the private key and save it in the mup config directory as `ssl.pem`. Check this [guide](http://alexnj.com/blog/configuring-a-positivessl-certificate-with-stud.html) to do that.

Then add following configuration to your `mup.json` file.

~~~js
{
  ...

  "ssl": {
    "pem": "./ssl.pem",
    //"backendPort": 80
  }

  ...
}
~~~

Now, simply do `mup setup` and now you've the SSL support.

> * By default, it'll think your Meteor app is running on port 80. If it's not, change it with the `backendPort` configuration field.
> * SSL terminator will run on the default SSL port `443`
> * If you are using multiple servers, SSL terminators will run on the each server (This is made to work with [cluster](https://github.com/meteorhacks/cluster))
> * Right now, you can't have multiple SSL terminators running inside a single server

### Updating

To update `mup` to the latest version, just type:

    npm update mup -g

You should try and keep `mup` up to date in order to keep up with the latest Meteor changes. But note that if you need to update your Node version, you'll have to run `mup setup` again before deploying.

### Troubleshooting

#### Check Logs
If you suddenly can't deploy your app anymore, first use the `mup logs -f` command to check the logs for error messages.

One of the most common problems is your Node version getting out of date. In that case, see “Updating” section above.

#### Verbose Output
If you need to see the output of `meteor-up` (to see more precisely where it's failing or hanging, for example), run it like so:

    DEBUG=* mup <command>

where `<command>` is one of the `mup` commands such as `setup`, `deploy`, etc.

### Binary Npm Module Support

Some of the Meteor core packages as well some of the community packages comes with npm modules which has been written in `C` or `C++`. These modules are platform dependent.
So, we need to do special handling, before running the bundle generated from `meteor bundle`.
(meteor up uses the meteor bundle)

Fortunately, Meteor Up **will take care** of that job for you and it will detect binary npm modules and re-build them before running your app on the given server.

> * Meteor 0.9 adds a similar feature where it allows package developers to publish their packages for different architecures, if their packages has binary npm modules.
> * As a side effect of that, if you are using a binary npm module inside your app via `meteorhacks:npm` package, you won't be able to deploy into `*.meteor.com`.
> * But, you'll be able to deploy with Meteor Up since we are re-building binary modules on the server.

### Additional Resources

* [Using Meteor Up with Nitrous.io](https://github.com/arunoda/meteor-up/wiki/Using-Meteor-Up-with-Nitrous.io)
* [Change Ownership of Additional Directories](https://github.com/arunoda/meteor-up/wiki/Change-Ownership-of-Additional-Directories)
* [Using Meteor Up with NginX vhosts](https://github.com/arunoda/meteor-up/wiki/Using-Meteor-Up-with-NginX-vhosts)
