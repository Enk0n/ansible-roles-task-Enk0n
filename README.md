# Ansible: app-for-devops (Apache + PostgreSQL, two hosts)

Deploys the Laravel app from
https://github.com/Practical-DevOps/app-for-devops on an Apache/PHP
server (`[server]` group) and configures a PostgreSQL server (`[db]` group)
on Ubuntu 22.04.

## Layout

```
main.yml                  # entry-point playbook
inventory.ini              # example inventory (edit hosts/IPs)
ansible.cfg
requirements.yml           # required collections
roles/
  db/                      # installs & configures PostgreSQL, creates DB/user
  server/                  # installs Apache/PHP/Node, deploys the app, wires .env
```

## 1. Install required collections

```bash
ansible-galaxy collection install -r requirements.yml
```

## 2. Edit the inventory

Set the real hosts under `[server]` and `[db]` in `inventory.ini`
(or supply your own inventory file with `-i`).

## 3. Run the playbook

```bash
ansible-playbook main.yml \
  --extra-vars "db_host=db.some.net db_name=app_db db_user=app_user db_pass=app_pass"
```

* `db_host` — hostname/IP the **application** uses to reach the database
  (normally the address of the `[db]` host).
* `db_name`, `db_user`, `db_pass` — database name and credentials created on
  the `[db]` host and written into the app's `.env` on the `[server]` host.

## What each role does

**`db`**
- Installs PostgreSQL and enables the service.
- Opens `listen_addresses = '*'` and adds a `pg_hba.conf` rule allowing the
  `[server]` host to connect over the network with `md5` auth.
- Creates the `db_user`/`db_pass` role and the `db_name` database, owned by
  that user, with full privileges granted.

**`server`**
- Installs Apache, PHP 8.1 + required extensions (`php-pgsql`, `mbstring`,
  `xml`, `curl`, `zip`, `bcmath`), Composer, and Node.js 18.x (via NodeSource,
  since Ubuntu 22.04's default `nodejs` package is too old to build the
  Vite front-end assets).
- Clones the application repository to `/var/www/app-for-devops`.
- Copies `.env.example` → `.env` and sets `DB_CONNECTION=pgsql`, `DB_HOST`,
  `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`.
- Runs `composer install`, `php artisan key:generate`, `php artisan migrate`,
  `npm install`, `npm run build`.
- Sets `www-data` ownership and writable permissions on `storage/` and
  `bootstrap/cache/`.
- Deploys an Apache vhost pointing `DocumentRoot` at `public/`, enables
  `mod_rewrite`, disables the default site, and (re)starts Apache.

## Notes

- Both roles assert that the required variables (`db_host`/`db_name`/
  `db_user`/`db_pass`) are defined and fail early with a clear message if not.
- `main.yml` opens with a `hosts: all` fact-gathering play. This guarantees
  `ansible_default_ipv4` is already known for the `[server]` host by the time
  the `db` role (which runs first) needs it to restrict `pg_hba.conf` to that
  specific IP, instead of opening PostgreSQL to the whole network.
- If `[server]` and `[db]` happen to point at the *same* machine (e.g. a
  local/CI smoke test where both groups list `localhost`), the default
  Ubuntu/Debian `pg_hba.conf` already ships a `127.0.0.1/32` rule, so the app
  can reach the database over `db_host=localhost` without any extra config.
- Node.js is installed from NodeSource rather than the Ubuntu 22.04 apt
  repo, because the distro's default `nodejs` package (v12) is too old to
  run this app's Vite-based `npm run build`. `nodejs_major_version` in
  `roles/server/defaults/main.yml` currently targets an actively supported
  LTS line — bump it there if that line goes end-of-life.
- If your CI only does `pip install ansible` (the full community package,
  as opposed to `ansible-core`), `community.general` and
  `community.postgresql` are already bundled — you don't need
  `requirements.yml` in that case. It's included here for setups that only
  have `ansible-core` installed.
