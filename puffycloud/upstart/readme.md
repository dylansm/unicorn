# Unicorn as a service using Upstart

Manage multiple Unicorn servers as services on the same box using Ubuntu upstart.

# Wait, what?

This is a copy-paste-tweak from the Puma project's [Jungle](https://github.com/puma/puma/tree/master/tools/jungle/upstart). I wanted to give Puma love, but Unicorn just seemed to work better for me.

One addition is the reliance on a blank flat file to determine Rails/Rack environment,
in order to support multistage deployments. For example, if the presence of YOUR_PROJECT/config/rack_env_staging
is detected (in /etc/init/unicorn.conf), either RAILS_ENV or DEPLOY_ENV will be set accordingly.

Note: DEPLOY_ENV is used instead of RACK_ENV because setting the -E flag to production in the start command overrides the RACK_ENV.
This way you can still run a staging environment under production.

## Installation 

    # Copy the scripts to services directory 
    sudo cp unicorn.conf unicorn-manager.conf /etc/init
    
    # Create an empty configuration file
    sudo touch /etc/unicorn.conf

## Managing the puffycloud

Unicorn apps are referenced in /etc/unicorn.conf by default. Add each app's path as a new line, e.g.:

```
/home/apps/my-cool-ruby-app
/home/apps/another-app/current
```

Start the puffycloud by running:

`sudo start unicorn-manager`

This script will run at boot time.

Start a single unicorn like this:

`sudo start unicorn app=/path/to/app`

## Logs

Everything is logged by upstart, defaulting to `/var/log/upstart`.

Each unicorn instance is named after its directory, so for an app called `/home/apps/my-app` the log file would be `/var/log/upstart/unicorn-_home_apps_my-app.log`.

## Conventions 

* The script expects:
  * a config file to exist under `config/unicorn.rb` in your app. E.g.: `/home/apps/my-app/config/unicorn.rb`.
  * a temporary folder to put the PID, socket and state files to exist called `tmp/unicorn`. E.g.: `/home/apps/my-app/tmp/unicorn`. Unicorn will take care of the files for you.

You can always change those defaults by editing the scripts.

## Example app config file.

Please note, the first few lines determine app names and paths based on DEPLOY_ENV or RAILS_ENV (if changed to RAILS_ENV)

```
env_apps_map = {
  staging: 'staging.my-app.com',
  production: 'my-app.com'
}

env = ENV['DEPLOY_ENV']
app_name = env_apps_map[env.to_sym]
@app_path = "/var/www/#{app_name}"

worker_processes 4
working_directory "#{@app_path}/current"

listen "#{@app_path}/tmp/unicorn/socket/#{env}.sock", backlog: 64
pid "#{@app_path}/tmp/unicorn/pid/#{env}.pid"
stderr_path "#{@app_path}/shared/log/unicorn.stderr.log"
stdout_path "#{@app_path}/shared/log/unicorn.stdout.log"
```

## Before starting...

You need to customise `unicorn.conf` to:

* Set the right user your app should be running on unless you want root to execute it!
  * Look for `setuid apps` and `setgid apps`, uncomment those lines and replace `apps` to whatever your deployment user is.
  * Replace `apps` on the paths (or set the right paths to your user's home) everywhere else.
* Uncomment the source lines for `rbenv` or `rvm` support unless you use a system wide installation of Ruby.
