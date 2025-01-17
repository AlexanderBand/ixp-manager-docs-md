# Upgrading IXP Manager - v5.x

???+ note "**These upgrade instructions relate to upgrading when you are already using IXP Manager v5.x.**"

    * For instructions on how to upgrade within the v4.x releases, please [see this page](upgrade-v4.md).

    * For instructions on how to upgrade from v3.x to v4, please [see this page](upgrade-v3.md).

    * For instructions on how to upgrade from v4.9.x to v5.0, please see [the v5.0 release notes](https://github.com/inex/IXP-Manager/releases).





We track [releases on GitHub](https://github.com/inex/IXP-Manager/releases).

You will find standard instructions for upgrading IXP Manager below. Note that the release notes for each version may contain specific upgrade instructions including schema changes.

If you have missed some versions, the most sensible approach is to upgrade to each minor release in sequence (5.0.0 -> 5.1.0 -> 5.2.0 -> ...) and then to the latest patch version in the latest minor version.

In the below, we assume the following installation directory - alter this to suit your own situation:

```
IXPROOT=/srv/ixpmanager
```


The general process is:

1. Set up some variables and ensure directory permissions are okay:

    ```sh
    # set this to your IXP Manager installation directory
    IXPROOT=/srv/ixpmanager

    # fix as appropriate to your operating system
    MY_WWW_USER=www-data

    # ensure the web server daemon user can write to necessary directories:
    chown -R $MY_WWW_USER: $IXPROOT/public/bower_components ${IXPROOT}/bower.json \
        ${IXPROOT}/storage $IXPROOT/vendor
    chmod -R u+rwX $IXPROOT/public/bower_components ${IXPROOT}/bower.json         \
        ${IXPROOT}/storage $IXPROOT/vendor
    ```

2. Enable maintenance mode:

    ```sh
    cd $IXPROOT
    php artisan down
    ```

3. Using Git, checkout the next minor / latest patch version up from yours. For IXP Manager v4.

    ```sh
    # (assuming we're still in $IXPROOT)
    # pull the latest code
    git fetch
    # check out the version you are upgrading to
    git checkout v5.x.y
    ```

4. Install latest required libraries from composer [**(see notes below)**](#updating-composer-dependancies):

    ```sh
    # this assumes composer.phar is in the IXP Manager install directory. YMMV - see notes below.
    sudo -u $MY_WWW_USER bash -c "HOME=${IXPROOT}/storage && cd ${IXPROOT} && php ./composer.phar install --no-dev --prefer-dist"
    ```

5. Restart Memcached and clear the cache. Do not forget / skip this step!

    ```sh
    # (assuming we're still in $IXPROOT)
    systemctl restart memcached.service
    ./artisan cache:clear
    ```

6. Update the database schema:

    ```sh
    # (assuming we're still in $IXPROOT)
    # (you really should take a mysqldump of your database first)
    # migrate:
    php artisan doctrine:schema:update --force
    php migrate
    ```

7. Restart Memcached (yes, again). Do not forget / skip this step!

    ```sh
    systemctl restart memcached.service
    ```

8. Ensure there are no version specific changes required in the release notes.

9. Ensure file permissions are still correct.

    ```sh
    chown -R $MY_WWW_USER: $IXPROOT/public/bower_components ${IXPROOT}/bower.json \
        ${IXPROOT}/storage $IXPROOT/vendor $IXPROOT/var $IXPROOT/bootstrap/cache
    chmod -R u+rwX $IXPROOT/public/bower_components ${IXPROOT}/bower.json         \
        ${IXPROOT}/storage $IXPROOT/vendor $IXPROOT/var $IXPROOT/bootstrap/cache
    ```

10. Clear out all caches:

    ```sh
    ${IXPROOT}/artisan cache:clear
    ${IXPROOT}/artisan config:clear
    ${IXPROOT}/artisan doctrine:clear:metadata:cache
    ${IXPROOT}/artisan doctrine:clear:query:cache
    ${IXPROOT}/artisan doctrine:clear:result:cache
    ${IXPROOT}/artisan route:clear
    ${IXPROOT}/artisan view:clear
    ```

11. Disable maintenance mode:

    ```sh
    # (assuming we're still in $IXPROOT)
    ./artisan up
    ```
12. Recreate SQL views

    Some older scripts, including the sflow modules, rely on MySQL view tables that may be affected by SQL updates. You can safely run this to recreate them:

    ```sh
    mysql -u ixp -p ixp < $IXPROOT/tools/sql/views.sql
    ```




## Updating Composer Dependancies

It is not advisable to run composer as root but how you run it will depend on your own installation. The following options would work on Ubuntu (run these as root and the composer commands themselves will be run as `$MY_WWW_USER`). Note that we assume here what you have installed Composer (see: https://getcomposer.org/ ) in the `${IXPROOT}` directory as `composer.phar`. This is where and how the IXP Manager installation scripts and documentation instructions install it.

```sh
# set this to your IXP Manager installation directory
IXPROOT=/srv/ixpmanager

MY_WWW_USER=www-data  # fix as appropriate to your operating system

# ensure www-data can write to vendor:
chown -R $MY_WWW_USER: $IXPROOT/vendor ${IXPROOT}/storage
chmod -R u+rwX $IXPROOT/vendor ${IXPROOT}/storage

# update composer
sudo -u $MY_WWW_USER bash -c "HOME=${IXPROOT}/storage && cd ${IXPROOT} && php ./composer.phar install"
```

NB: If composer is not managed by your package management system, you should keep it up to date via the following (using the same definitions from the composer update example above):

```sh
chown -R $MY_WWW_USER: ${IXPROOT}/composer.phar
chmod -R u+rwx ${IXPROOT}/composer.phar
sudo -u $MY_WWW_USER bash -c "HOME=${IXPROOT}/storage && cd ${IXPROOT} && php ./composer.phar selfupdate"
```



## Correcting Database Issues / Verifying Your Schema

Because of the manual process of database updates, it is possible your database schema may fall out of sync.

If you are having issues, first and foremost, restart Memcached. Doctrine2 caches entities and schema information in Memcached so, after an upgrade, you must restart Memcached.

You can verify and update your schema using the `artisan` script. The first action should be validation - here is a working example with no database issues:

```
cd $IXPROOT
./artisan doctrine:schema:validate

Validating for default entity manager...
[Mapping]  OK - The mapping files are correct.
[Database] OK - The database schema is in sync with the mapping files.
```

If there are issues, you can use the following to show what SQL commands are required to bring your schema into line:

```
./artisan doctrine:schema:update --sql
```

And you can let Doctrine make the changes for you via:

```
./artisan doctrine:schema:update --force
```

Doctrine2 maintains the entities, proxies and repository classes. Ideally you should never need to do the following on a production installation - as we maintain these files with Git - but if you're developing / testing IXP Manager, you may need to.

The process for updating these files with schema changes / updates is:

```
cd $IXPROOT
systemctl restart memcached.service           # (or as appropriate for your system)
./artisan doctrine:generate:entities database
./artisan doctrine:generate:proxies
```
