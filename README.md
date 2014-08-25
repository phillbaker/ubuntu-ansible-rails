# Ubuntu Rails

This will provision an Ubuntu Rails box suitable for production use running Nginx, Unicorn and (optionally a db)
 on a cloud Ubuntu server or local Vagrant environment that can be deployed to via capistrano.

It should allow you to have a development environment that is identical to your production environment.

Note this prompts for a dpeloy user password, however, access to the deploy user is only allowed via public keys.

## Stack

+ Nginx
+ Unicorn
+ Rbenv
+ (Node.js)
+ (Postgres or MySQL)

## Use

After setup, use Capistrano to deploy your web application to your Vagrant box or cloud server.

```
git clone git@github.com:phillbaker/ubuntu-ansible-rails.git
cd ubuntu-ansible-rails
cp vagrant-hosts.example vagrant-hosts
```

### Production

For production, create your cloud box with your ssh key already on the root user. Update `vagrant-hosts` with the IP address. Then run:

```
./deploy.sh
```

### Development

For local Vagrant development

```
vagrant up
./deploy.sh
ssh deploy@33.33.33.3
```

## Manual follow up

* TODO move this into templates

chown -R deploy:deploy /home/deploy/.rbenv

Create & set file permissions on /var/www

```
sudo mkdir -p /var/www
sudo adduser deploy www-data
sudo chown -R www-data:www-data /var/www
sudo chmod -R g+rw /var/www
```

Configure nginx

```
sudo ln -s /var/www/sitename/current/config/nginx.conf /etc/nginx/sites-enabled/sitename.com
sudo ln -s /var/www/sitename/current/config/nginx.conf /etc/nginx/sites-available/sitename.com
```

or

```
sudo vim /etc/nginx/sites-available/sitename.com.nginx.conf
sudo ln -s /etc/nginx/sites-available/pharmacy.io.nginx.conf /etc/nginx/sites-enabled/pharmacy.io.nginx.conf
sudo service nginx restart

cd /var/www/sitename/current
rbenv exec bundle exec unicorn -c config/unicorn.rb -D -E production
```

Nginx config

```nginx
upstream unicorn {
  server unix:/var/www/sitename/current/tmp/sockets/unicorn.sock fail_timeout=0;
}

server {
  listen 80 default deferred;
  root /var/www/sitename/current/public;

  try_files $uri/maintenance.html $uri/index.html $uri @unicorn;

  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unicorn;
  }

  location ~ ^/(assets)/  {
    root /var/www/sitename/current/public;
    gzip_static on;
    expires     max;
    add_header  Cache-Control public;
  }

  location = /favicon.ico {
    expires    max;
    add_header Cache-Control public;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 4G;
  keepalive_timeout 10;
}
```

init.d script

```
sudo vim /etc/init.d/unicorn-sitename.com
```

```
#!/bin/sh
set -e
TIMEOUT=${TIMEOUT-60}
APP_ROOT=/var/www/sitename.com/current
PID=$APP_ROOT/tmp/pids/unicorn.pid
CMD="bundle exec unicorn_rails -D -c $APP_ROOT/config/unicorn.rb -E production"
action="$1"
set -u
old_pid="$PID.oldbin"
cd $APP_ROOT || exit 1
sig () {
        test -s "$PID" && kill -s $1 `cat $PID`
}
oldsig () {
        test -s $old_pid && kill -s $1 `cat $old_pid`
}
case $action in
start)
        sig 0 && echo >&2 "Already running" && exit 0
        $CMD
        ;;
stop)
        sig QUIT && exit 0
        echo >&2 "Not running"
        ;;
force-stop)
        sig TERM && exit 0
        echo >&2 "Not running"
        ;;
restart|reload)
        sig HUP && echo reloaded OK && exit 0
        echo >&2 "Couldn't reload, starting '$CMD' instead"
        $CMD
        ;;
upgrade)
        if sig USR2 && sleep 5 && sig 0 && oldsig QUIT
        then
                n=$TIMEOUT
                while test -s $old_pid && test $n -ge 0
                do
                        printf '.' && sleep 1 && n=$(( $n - 1 ))
                done
                echo
                if test $n -lt 0 && test -s $old_pid
                then
                        echo >&2 "$old_pid still exists after $TIMEOUT seconds"
                        exit 1
                fi
                exit 0
        fi
        echo >&2 "Couldn't upgrade, starting '$CMD' instead"
        $CMD
        ;;
reopen-logs)
        sig USR1
        ;;
*)
        echo >&2 "Usage: $0 <start|stop|restart|upgrade|force-stop|reopen-logs>"
        exit 1
        ;;
esac
```




Unicorn.rb (config/unicorn.rb)

```ruby
# SET YOUR HOME DIR HERE

APP_ROOT = File.expand_path(File.dirname(File.dirname(__FILE__)))

worker_processes 16
working_directory APP_ROOT
preload_app true
timeout 30
rails_env = ENV['RAILS_ENV'] || 'production'

listen APP_ROOT + "/tmp/sockets/unicorn.sock", :backlog => 64
pid APP_ROOT + "/tmp/pids/unicorn.pid"

stderr_path APP_ROOT + "/log/unicorn.stderr.log"
stdout_path APP_ROOT + "/log/unicorn.stdout.log"

before_fork do |server, worker|
  ActiveRecord::Base.connection.disconnect!

  old_pid = APP_ROOT + '/tmp/pids/unicorn.pid.oldbin'
  if File.exists?(old_pid) && server.pid != old_pid
    begin
      Process.kill("QUIT", File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
      puts "Old master already dead"
    end
  end
end

after_fork do |server, worker|
  ActiveRecord::Base.establish_connection
end
```
