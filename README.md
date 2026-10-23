# Heroku Buildpack: heroku-db-backup-s3
Capture Postgres or MySQL DB in Heroku, encrypt it with GPG and copy it to s3 bucket. Buildpack contains AWS CLI.

## Installation

### Configure environment variables
```
$ heroku config:add DB_BACKUP_FILENAME=client_name-postgres --app <your_app>
$ heroku config:add DB_BACKUP_AWS_ACCESS_KEY_ID=someaccesskey --app <your_app>
$ heroku config:add DB_BACKUP_AWS_SECRET_ACCESS_KEY=supermegasecret --app <your_app>
$ heroku config:add DB_BACKUP_AWS_DEFAULT_REGION=eu-central-1 --app <your_app>
$ heroku config:add DB_BACKUP_S3_BUCKET_PATH=bucket-name --app <your_app>
$ heroku config:add DB_BACKUP_GPG_PUB_KEY=public_key_armor_export --app <your_app>
$ heroku config:add DB_BACKUP_GPG_PUB_KEY_ID=gpg_recipient --app <your_app>
```
It uses `DATABASE_URL` as the database to be backed up.

See below how to [create the GPG key](#creating-gpg-key)

### Using an S3-compatible provider (e.g. Scaleway)

Any S3-compatible object storage can be used instead of AWS by setting `DB_BACKUP_S3_ENDPOINT_URL`. The `DB_BACKUP_AWS_*` variables then hold the provider's credentials and region.

For [Scaleway Object Storage](https://www.scaleway.com/en/docs/object-storage/), create an API key in the Scaleway console (IAM → API keys) and use the region of your bucket (`fr-par`, `nl-ams` or `pl-waw`):
```
$ heroku config:add DB_BACKUP_AWS_ACCESS_KEY_ID=SCWXXXXXXXXXXXXXXXXX --app <your_app>
$ heroku config:add DB_BACKUP_AWS_SECRET_ACCESS_KEY=scaleway_secret_key --app <your_app>
$ heroku config:add DB_BACKUP_AWS_DEFAULT_REGION=fr-par --app <your_app>
$ heroku config:add DB_BACKUP_S3_ENDPOINT_URL=https://s3.fr-par.scw.cloud --app <your_app>
$ heroku config:add DB_BACKUP_S3_BUCKET_PATH=bucket-name --app <your_app>
```
The region in `DB_BACKUP_AWS_DEFAULT_REGION` must match the region in the endpoint URL.

### Add buildpack to your Heroku app
```
heroku buildpacks:add --index 1 https://github.com/abtion/heroku-db-backup-s3 --app <your_app>
```

### For Postgres:

Works out of the box

### For MySQL:

You will need to install a mysql buildpack to make the `mysqldump` command available. For example:

```
$ heroku buildpacks:add --index 1 https://github.com/heroku/heroku-buildpack-apt --app <your_app>
```
In your project root file

```
$ echo "mysql-client" > Aptfile
```

### Scheduler
Add addon scheduler to your app.
```
$ heroku addons:create scheduler --app <your_app>
```
Create scheduler.
```
$ heroku addons:open scheduler --app <your_app>
```
Now in browser `Add new Job`.
Paste next line:
`bash /app/vendor/backup.sh`
and configure FREQUENCY.

## One-time runs

You can run the backup task as a one-time task:

```
$ heroku run bash /app/vendor/backup.sh --app <your_app>
```

## Doesn't work?
In case if scheduler doesn't run your task, check logs using this e.g.:
```
$ heroku logs -t  --app <your_app> | grep 'backup.sh'
$ heroku logs --ps scheduler.x --app <you_app>
```

## Restoring

Remember to delete the secret key again after use, both the temporary file for importing and from GPG `gpg --delete-secret-keys <key_id>`

Save the secret key to a file.
Import it `gpg --import <secret_key_file_name>`

```bash
$ gpg --decrypt <encrypted_file>
```

## Creating GPG key
```
$ gpg --full-generate-key
```

Generate a strong passphrase for the gpg private key.
Choose the defaults (make sure your version of gpg is updated).
Store the private key and passphrase in a secured vault.

List the key.
```
$ gpg --list-keys
```

Copy the key id (looks something like 16257D612F99FC6A197638635A934E07E658EA2).
Export the secret key.
```
$ gpg --armor --export-secret-key <key_id>
```

Export the public key.
```
$ gpg --armor --export <key_id>
```

Copy your GPG key, beginning with -----BEGIN PGP PUBLIC KEY BLOCK----- and ending with -----END PGP PUBLIC KEY BLOCK-----.
Create the DB_BACKUP_GPG_PUB_KEY and DB_BACKUP_GPG_PUB_KEY_ID config vars.

Delete the private key from your system
```
$ gpg --delete-secret-keys <key_id>
```
