# Run PostgreSQL with systemd and `pg_ctl`

`postgresql-18@.service` runs an explicitly initialized cluster using `pg_ctl`. An instance such as `postgresql-18@main.service` uses `main` as its directory name. An instance named `main` uses a single directory named `main`; the name does not select a PostgreSQL version.

The paths use [systemd's directory specifiers](https://raw.githubusercontent.com/systemd/systemd/main/man/systemd.unit.xml):

| Purpose                   | Unit path          | System service             | User service default               |
| ------------------------- | ------------------ | -------------------------- | ---------------------------------- |
| Configuration             | `%E/postgresql/%i` | `/etc/postgresql/main`     | `~/.config/postgresql/main`        |
| Database state (`PGDATA`) | `%S/postgresql/%i` | `/var/lib/postgresql/main` | `~/.local/state/postgresql/main`   |
| Unix socket               | `%t/postgresql/%i` | `/run/postgresql/main`     | `$XDG_RUNTIME_DIR/postgresql/main` |

User paths honor `XDG_CONFIG_HOME`, `XDG_STATE_HOME`, and `XDG_RUNTIME_DIR` in the **user service manager's environment**. Configure custom XDG roots before starting the user manager; setting them only in a shell or the unit's `Environment=` does not change its directory specifiers. The system config root is `/etc`, following systemd's system-service convention.

## Set up a user service

1. Install PostgreSQL server binaries. The template explicitly uses
   `/usr/bin/pg_ctl`. If your installation uses another directory, replace that path in **all three** `ExecStart`, `ExecStop`, and `ExecReload` commands in the installed unit. Use `initdb` from the same installation and a PostgreSQL major version compatible with the cluster.

2. From the repository root, install the template and initialize a **new** cluster as the user who will run it. Adjust `/usr/bin/initdb` if needed:

   ```sh
   pg_config_dir="${XDG_CONFIG_HOME:-$HOME/.config}/postgresql/main"
   pg_data_dir="${XDG_STATE_HOME:-$HOME/.local/state}/postgresql/main"
   pg_unit_dir="${XDG_CONFIG_HOME:-$HOME/.config}/systemd/user"

   install -d -m 0700 "$pg_config_dir" "$pg_data_dir" "$pg_unit_dir"
   install -m 0644 deployments/postgresql/postgresql-18@.service "$pg_unit_dir/postgresql-18@.service"
   /usr/bin/initdb --pgdata "$pg_data_dir" --auth-local=peer --auth-host=scram-sha-256
   mv "$pg_data_dir/postgresql.conf" "$pg_data_dir/pg_hba.conf" "$pg_data_dir/pg_ident.conf" "$pg_config_dir/"
   ```

   For an existing cluster, stop its current service first, use its existing data directory or move it to the path above, and move the configuration files separately. Do not run `initdb` on an existing cluster. Update any configuration includes that depend on the old location.

3. Edit the external `postgresql.conf` and `pg_hba.conf` for your deployment. Give concurrently running clusters distinct ports. The unit passes `data_directory`, `config_file`, `hba_file`, `ident_file`, and `unix_socket_directories` explicitly; change the unit if those paths need to change. PostgreSQL supports these [separate configuration locations](https://www.postgresql.org/docs/current/runtime-config-file-locations.html). Its generated [`postgresql.auto.conf`](https://www.postgresql.org/docs/current/config-setting.html#CONFIG-SETTING-CONFIGURATION-FILE) remains in `PGDATA` for `ALTER SYSTEM`.

4. Verify and start the service:

   ```sh
   systemd-analyze --user verify "$pg_unit_dir/postgresql-18@.service"
   systemctl --user daemon-reload
   systemctl --user enable --now postgresql-18@main.service
   systemctl --user status postgresql-18@main.service
   journalctl --user -u postgresql-18@main.service
   ```

   The service creates its socket directory with mode `0700`. Connect as the cluster owner with `psql --host "$XDG_RUNTIME_DIR/postgresql/main" postgres`, or configure TCP authentication. Enable lingering for the account with `loginctl enable-linger USER` if the service must run without a login session.

## Set up a system service

Install the same template at `/etc/systemd/system/postgresql-18@.service` and
adjust its three `pg_ctl` paths if needed. Before enabling it, create
`/etc/systemd/system/postgresql-18@.service.d/system.conf`:

```ini
[Service]
User=postgres
Group=postgres

[Install]
WantedBy=multi-user.target
```

Create the `postgres` account if your PostgreSQL installation has not already done so. Initialize `/var/lib/postgresql/main` as that account with mode `0700`, then move `postgresql.conf`, `pg_hba.conf`, and `pg_ident.conf` into `/etc/postgresql/main`. Ensure the account can traverse the config directory and read those files. Use the initialization and authentication flags above with the system paths. PostgreSQL must run as the cluster owner, not root.

```sh
sudo systemd-analyze verify /etc/systemd/system/postgresql-18@.service
sudo systemctl daemon-reload
sudo systemctl enable --now postgresql-18@main.service
sudo systemctl status postgresql-18@main.service
sudo journalctl -u postgresql-18@main.service
```

## Operate the cluster

- Use `systemctl --user reload`, `restart`, or `stop` with the instance name;
- use `sudo systemctl` for a system service. Reload sends PostgreSQL a request to reread configuration; inspect its logs for rejected settings or settings that require a restart.

[`pg_ctl --wait`](https://www.postgresql.org/docs/current/app-pg-ctl.html) waits for startup readiness and shutdown completion. Both operations have a one-hour limit, with one extra minute allowed by systemd. Increase both the relevant `--timeout` and `TimeoutStartSec` or `TimeoutStopSec` if needed for long recovery or shutdown. Startup errors fail the unit. Shutdown uses `fast` mode, rolling back active transactions and disconnecting clients cleanly. Automatic service restarts are disabled. Output goes to the journal unless PostgreSQL's own logging configuration redirects it.
