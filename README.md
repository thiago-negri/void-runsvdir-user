void-runsvdir-user
==================

User services on Void Linux (runit).


Sinopsis
--------

Users can define services to run under their user accounts.  Services will still start at boot, no need to start a login
shell as the user.


How?
----

Users should define their services in `~/service` path, as follows:

1. `~/service/sv`: this is the folder that runit will monitor via `runsvdir`, so it has the same properties as
   `/var/service` and services should be configured the same way.
2. `~/service/dependencies`: text file that list service names that this user depends on, the system service will
   wait until all those services are up before executing `runsvdir` on `~/service/sv`.
3. `~/service/log`: user services will have their outputs routed here.


Installing
----------

Run `./install` as root to:

1. Create a custom `/etc/vlogger` that will route user services logs to their home `service/log` folder.
   System service logs will be routed to `/var/log`.  Why?  Because services running as a normal user usually has no
   permission to write to `/var/log`.
2. Create a `runsvdir-{user}` service for each regular user in that machine that has a `~/service` folder.  That
   service will automatically start `runsvdir` on their `~/service/sv` after their defined dependencies are up.
3. Create a shell profile extension that sets `SVDIR` for users that have a `~/service` folder.  That way they can
   use `sv` commands to query the services running under their name.

If you create new users, you will need to run `./install` again to create services for those users.

By default the `install` script will **not** overwrite any file, if you really want to overwrite stuff, run it with
`-f` option, like `./install -f`.

As you need to run that as root, please audit the script to know what it's doing, it's pretty small.


Uninstalling
------------

This is a list of files/folders that this script will touch in your system.  You may delete them if you're sure you
don't use them for other stuff, for example the `/etc/vlogger` may have been changed by other stuff.  In doubt,
compare it with their origin in this repository to see if they still match.

```
/etc/vlogger
/etc/sv/runsvdir-*
/var/service/runsvdir-*
/etc/profile.d/runsvdir-user.sh
```


Example (PipeWire)
------------------

As an example (i.e. the main reason I created this), we can have runit start PipeWire as a regular user.

1. Login as the regular user.

2. Create the service folders.
```sh
mkdir -p ~/service/sv
```

3. Create the PipeWire service definition.
```sh
mkdir -p ~/service/def/pipewire/log

tee ~/service/def/pipewire/run <<-EOF
#!/bin/sh
exec 2>&1
XDG_RUNTIME_DIR="/run/user/$(id -u)" exec pipewire
EOF

tee ~/service/def/pipewire/log/run <<-EOF
#!/bin/sh
exec vlogger -t pipewire -p daemon
EOF

chmod +x ~/service/def/pipewire/run ~/service/def/pipewire/log/run
```

4. List the dependencies (services you want to wait that they are up before starting your own services).
```sh
tee ~/service/dependencies <<-EOF
alsa
elogind
dbus
EOF
```

5. Run the install script of this repository as root.
```sh
sudo ./install
```

6. Link the service definition so runit starts it.
```sh
ln -s ~/service/def/pipewire ~/service/sv/pipewire
```

If everything goes well, the end result will be:

1. System service `runsvdir-$USER` that execs into `runsvdir` as `$USER`.
2. `pipewire` service running as `$USER` because the previous `runsvdir` started it.
3. PipeWire logs will be available at `~/service/log/pipewire`.

If you don't see the logs, try restarting all service logs (the install script changed `/etc/vlogger`):
```sh
sudo sv restart /var/service/*/log
sv restart pipewire/log
```

If the `sv restart ...` command fails, open a new shell or prefix your command with `SVDIR` definition, like so:
```sh
SVDIR=~/service/sv restart pipewire/log
```

The `SVDIR` var will be set automatically for your user on new shells (thanks to the install script).

If it still doesn't show up, maybe your user session already has PipeWire running?  Make sure you're not starting
it as part of X (check your `~/.xinitrc`).  To be sure everything works the way it should, make sure you reboot
your system and see that PipeWire is started during boot correctly.  It may emit some warnings that there is no X
running when it starts, but it should work fine when you start X, no need to restart any service.
