# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'active_support/core_ext/hash/deep_merge'

Vagrant.configure("2") do |config|

  config.berkshelf.enabled = true
  config.vm.box = "ubuntu-12.04-omnibus-chef"
  config.vm.box_url = "https://s3.amazonaws.com/gsc-vagrant-boxes/ubuntu-12.04-omnibus-chef.box"

  # http://bartek.im/blog/2012/12/04/postgresql-92-streaming-primer.html
  postgresql = {
    "version" => '9.2',
    "conf" => {
      "data_directory" => '/var/lib/postgresql/9.2/main',
      "hba_file" =>'/etc/postgresql/9.2/main/pg_hba.conf',
      "ident_file" => '/etc/postgresql/9.2/main/pg_ident.conf',
      "external_pid_file" => '/var/run/postgresql/9.2-main.pid'  ,
      "listen_addresses" => '*',
      "port" => 5432,
      "max_connections" => 100,
      "unix_socket_directory" => '/var/run/postgresql',
      "ssl" => true,
      "ssl_cert_file" => '/etc/ssl/certs/ssl-cert-snakeoil.pem',
      "ssl_key_file" => '/etc/ssl/private/ssl-cert-snakeoil.key',
      "shared_buffers" => '24MB',
      "wal_level" => "hot_standby",
      "max_wal_senders" => 5    ,
      "wal_keep_segments" => 32,
      "synchronous_standby_names" => '*',
      "log_line_prefix" => '%t ',
      "datestyle" => 'iso, mdy',
      "lc_messages" => 'en_US.UTF-8',
      "lc_monetary" => 'en_US.UTF-8',
      "lc_numeric" => 'en_US.UTF-8',
      "lc_time" => 'en_US.UTF-8'
    },
    "users" => [
      {
        "username" => "postgres",
        "password" => "password",
        "superuser" => true,
        "createdb" => true,
        "login" => true
      },
      {
        "username" => "epg",
        "password" => "epg",
        "createdb" => true,
        "login" => true
      },
      {
        "username" => "replicator",
        "password" => "replicator",
        "login" => true
      }
    ],
    "pg_hba" => [
      { :type => "local", :db => "all", :user => "postgres",   :addr => "",          :method => "ident" },
      { :type => "local", :db => "all", :user => "all",        :addr => "",          :method => "trust" },
      { :type => "host",  :db => "all", :user => "all",        :addr => "0.0.0.0/0", :method => "trust" }, # FIXME
      { :type => "host",  :db => "all", :user => "all",        :addr => "::1/0",     :method => "trust" }, # FIXME
      #  { :type => "host",  :db => "all", :user => "postgres",   :addr => "127.0.0.1/32", :method => "trust" },
      #  { :type => "host",  :db => "  all", :user => "epg",        :addr => "127.0.0.1/32", :method => "trust" },
      { :type => "host",  :db => "replication", :user => "replicator", :addr => "127.0.0.1/32", :method => "trust" },
      { :type => "local",  :db => "replication", :user => "postgres", :method => "trust" }
    ]
  }

  config.vm.define :master do | master_config|
    master_config.vm.network :private_network, ip: "33.33.33.72"
    master_config.vm.hostname = "dbmaster"
    master_config.vm.provision :chef_solo do |chef|
      chef.log_level = :debug
      chef.add_recipe("postgresql::server")

      chef.json["postgresql"] = postgresql.deep_merge({
        "conf" => {
          "archive_mode" => "on",
          "archive_command" => 'rsync -aq %p postgres@slave1.example.internal:/var/lib/postgresql/9.2/archive/%f',
          "archive_timeout" => 3600
        },
        "databases" => [
          {
            "name" => "schedule-data",
            "owner" => "epg",
            "encoding" => "utf8",
            "locale" => "en_US.utf8"
          }
        ]
      })
      puts "hash: #{chef.json}"
    end
  end

  config.vm.define :slave do |slave_config|
    slave_config.vm.network :private_network, ip: "33.33.33.73"
    slave_config.vm.hostname = "dbslave"
    slave_config.vm.provision :chef_solo do |chef|
      chef.log_level = :debug
      chef.add_recipe("postgresql::server")

      chef.json["postgresql"] = postgresql.deep_merge({
        "conf" => {
          "hot_standby" => "on"
        }
      })
    end
  end

  #standby_mode = 'on' # enables stand-by (readonly) mode
  #
  ## Connect to the master postgres server using the replicator user we created.
  #primary_conninfo = 'host=pgmaster.mysite.internal port=5432 user=replicator'
  #
  ## Specifies a trigger file whose presence should cause streaming replication to
  ## end (i.e., failover).
  #trigger_file = '/tmp/pg_failover_trigger'
  #
  ## Shell command to execute an archived segment of WAL file series.
  ## Required for archive recovery if streaming replication falls behind too far.
  #restore_command = 'cp /var/lib/postgresql/9.2/archive/%f %p'
  #archive_cleanup_command = 'pg_archivecleanup /var/lib/postgresql/9.2/archive/ %r'

end