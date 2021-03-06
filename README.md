NOTE
====

This is a fork (converted to git) from https://bitbucket.org/winebarrel/ruby-binlog

Description
===========

ruby-binlog is Ruby binding for MySQL Binary log API.

* http://www.oscon.com/oscon2011/public/schedule/detail/18785
* https://launchpad.net/mysql-replication-listener
  * https://bitbucket.org/winebarrel/mysql-replication-listener

Install
=======

gem install ruby-binlog

Required Privileges
===================

* SUPER
* REPLICATION SLAVE
* EVENT

Example
=======

``` ruby
  #!/usr/bin/env ruby
  require "rubygems"
  require "binlog"
  require "pstore"
  
  $db = PStore.new("/tmp/foo")
  
  #master_log_file = "mysql-bin.000001"
  #master_log_pos = 4
  
  master_log_file = nil
  master_log_pos = nil
  
  $db.transaction do
    master_log_file = $db["master_log_file"]
    master_log_pos = $db["master_log_pos"]
  end
  
  def save_position(master_log_file, master_log_pos)
    $db.transaction do
      $db["master_log_file"] = master_log_file
      $db["master_log_pos"] = master_log_pos
    end
  end
  
  begin
    # XXX: Do not reuse a client instance, after connection goes out.
    client = Binlog::Client.new("mysql://repl:repl@example.com")
    sleep 0.3 until client.connect
  
    if master_log_file and master_log_pos
      client.set_position(master_log_file, master_log_pos)
    elsif master_log_pos
      client.position = master_log_pos
    end
  
    while event = client.wait_for_next_event
      puts "(#{event.event_type})"
      master_log_pos = event.next_position
  
      case event
      when Binlog::QueryEvent
        puts event.db_name
        puts event.query
        save_position(master_log_file, master_log_pos)
      when Binlog::RowEvent
        puts event.event_type
        puts event.db_name
        puts event.table_name
        p event.columns
        p event.rows
        save_position(master_log_file, master_log_pos)
      when Binlog::RotateEvent
        master_log_file = event.binlog_file
        master_log_pos = event.binlog_pos
        save_position(master_log_file, master_log_pos)
      end
    end
  rescue Binlog::Error => e
    puts e
    retry if client.closed?
    raise e
  end
```

Notice
======

The following type are not supported in row mode.

* ENUM
* SET
* GEOMETRY
