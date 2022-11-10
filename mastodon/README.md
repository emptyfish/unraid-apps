# wonderfall-mastodon Unraid container

This is a companion to the [wonderfall-mastodon](https://github.com/Wonderfall/docker-mastodon) Community Application published on the Unraid application store. Credit to this post and this container setup goes to FixFlare and the original documentation provided here: [Mastodon on Unraid](https://fixflare.com/en/posts/unraid/mastodon/)

This container allows users to [run their own Mastodon instance](https://docs.joinmastodon.org/user/run-your-own/)

Please review these details carefully - this container (and Mastodon in general) require multiple services to function. This container really just saves you the time of manually adding the variables required for the container to operate. This guide details what specifically worked for my environment; your experience may vary depending on what versions each service you are using, your hardware, etc.

One last recommendation: start by setting up an instance for your own use only. Turn off registrations after you set up your admin account. You can always open registration later - but learning how to manage an instance of Mastodon may take some time. Understanding how moderation, blocking, etc. work and will help you. Great power/responsibility and all that... There's also understanding what kind of user capacity and storage your container can handle. After just 5 days I have about 2.5GB of media storage in use.

## Prerequisites

Before you install Mastodon you will need:

- A **domain/subdomain** that you can point to the container. Mastodon will not work from a local IP.
- A **reverse proxy** solution - I use SWAG and set up via NGINX but any similar solution will work.

You will also need the following containers/apps working:

- **[Postgres](https://unraid.net/community/apps?q=postgres#r)** for the database; I used postgres:15-alpine edition and have no issues.
- **[Redis](https://unraid.net/community/apps/c/network?q=redis#r)** for cache operations
- An **SMTP API account or a container capable of sending emails**. I have published a Community Applications version of [Bytemark SMTP](https://unraid.net/community/apps?q=bytemark-smtp) that can be used for internal email. The instructions below show the configuration working.

### Database setup

In my postgres container I manually created a user and database for mastodon, both called **mastodon** prior to setup. Create the user first, then the database and make sure the user has full permissions on the database.

The one additional step to perform is to enable connections from the local docker IP range so the Mastodon container is able to connect to the database. You can do this by editing the file `pg_hba.conf` in the postgres container - add the following:

```bash
# TYPE  DATABASE USER  ADDRESS   METHOD

host mastodon mastodon x.x.x.0/16 trust
```

*replace the "x" values shown above with your container subnet if using this approach, ex. `198.76.5.0/16`

Assuming you have covered the database, redis, SMTP and subdomain/proxy steps, installing the mastodon application is relatively simple. Pull the container from the Community Apps store on Unraid - alternately you can manually create the container using the variables provided below:

| Variable | Example/Value | Notes |
| --- | --- | --- |
| Repository | `ghcr.io/wonderfall/mastodon` | This is the current location of this container |
| Mastodon Path | | Select a local path; will be mapped to `/mastodon/public/system` in the container |
| LOCAL_DOMAIN | `mastodon.yourdomain.org` | **Important** - changing this value can break your instance per the Mastodon documentation |
| OTP_SECRET | | **Leave blank** when creating the container; see below for instructions |
| SECRET_KEY_BASE | | **Leave blank** when creating the container; see below for instructions |
| VAPID_PRIVATE_KEY | | **Leave blank** when creating the container; see below for instructions |
| VAPID_PUBLIC_KEY | | **Leave blank** when creating the container; see below for instructions |
| Web Port | `3000` | Select an available port in your Unraid server; will be mapped to port 3000 in the container |
| Streaming Port | `4000` | Select an available port in your Unraid server; will be mapped to port 4000 in the container |
| DB_HOST | `mastodon-postgres` | should container name or address of postgres server |
| DB_NAME | `mastodon` | should container name or address of postgres server |
| DB_USER | `mastodon` | should match user created in your postgres DB |
| DB_PASS | | should match user password created in your postgres DB |
| REDIS_HOST | `mastodon-redis` | container name or address of your redis server |
| SMTP_SERVER | `mastodon-smtp` | Example assumes using the Bytemark smtp container and matches the container name; you can use any SMTP service as well |
| SMTP_PORT | `25` | SMTP port; default is 25 |
| SMTP_LOGIN | | Leave blank if using the Bytemark SMTP server |
| SMTP_PASSWORD |  | Leave blank if using the Bytemark SMTP service |
| SMTP_FROM_ADDRESS | `youremail@yourdomain.org` | Any mail sent from Mastodon will show this as sender address, important that this is a working email account you have access to |

Once you have confirmed all of the values in the Mastodon container go ahead and save/start the container. A couple of quick things to check once the container starts:

- Check the log on your postgres instance to make sure there are no connection errors being reported. I did not set up the host correctly on my first attempt and I was able to identify the issue by reviewing the logs from the container.
- Also check the redis container for any log/errors.
- Finally check to confirm the new container started and is running/no errors were reported.

### Generating keys

Assuming the container is running you can now create the required keys for the container by opening the container console - enter the following:

```bash
bundle exec rake secret
```

You should get a response in the command line with a key generated. Copy/paste this value into the **OTP_SECRET** variable in the container.

Repeat the same command again:

```bash
bundle exec rake secret
```

The response should generate another unique value; copy/paste this value into the **OTP_SECRET_KEY_BASE** field.

In the console enter the next command:

```bash
bundle exec rake mastodon:webpush:generate_vapid_key
```

The output of this command should provide you with two values; enter them into VAPID_PRIVATE_KEY and VAPID_PUBLIC_KEY respectively.

Once all four values have been populated, save/restart your container. At this point you should now be able to launch a web browser and go to `example.yourdomain.org` and you should see the main Mastodon page:

(screenshot)

Registration should currently be open; enter whatever username/password/email information you want to use for your administrator account and submit your new account registration. The email address you provide needs to work - you will receive a confirmation to that email address to confirm your account - follow those steps.

From the container console, you can now enable your account to have administration rights. Open the container console and enter the following (replace username with the username you registered as):

```bash
tootctl accounts modify username --role admin
```

Once the command completes you should be able to sign in via the web interface. Select **Preferences** from the right side of the page and then look for the **Administration** section. Under **Site settings** you can change the registration mode to *Nobody can sign up*; this will prevent users from registering while you configure your environment to your preferences. There are many other options - see [this guide](https://docs.joinmastodon.org/admin/setup/) for more details.
