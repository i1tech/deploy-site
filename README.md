deploy-site for Drupal
======================

The `deploy-site` script is used by Classic to manage the test, staging, and live deployment processes on shared Ubuntu virtual machines. It depends on a Github server, either github.com or the enterprise edition.

Using the `deploy-site` script as the only method for getting new code on to a web virtual machine helps can help ensure only reviewed code goes to any of the servers. If users have write access to a deployed repository, the `deploy-site` script also pushes a deploy tag back to the originating repository. For environments with multiple users, running `deploy-site --deployed` will show the details on the latest code deployed to the server.

```
Current Deploy Information for this server:

     Deploying user: 
     Drupal Version: 
        Environment: 
       Tag Deployed: 
    Branch Deployed: 
         Repository: 
GitHub Organization: 
```

Each deploy also creates a database backup. The backups are stored as `/var/dbbackup/${ENVIRON}-{TAG}.sql` using Drush.

## Server-side configuration

Running `deploy-site` is automatically setup to use the hostname of the Ubuntu server to locate the matching the repository on Github. Hostnames and repositories are standardized by customer.

Take the Big Tube Company as an example, a fictional company. They host their website at http://bigtubeco.com. Their staging and test site FQDN are staging.bigtubeco.com and test.bigtubeco.com. The hostname of their production Ubuntu VM would be named `bigtubeco`, the staging server hostname `bigtubeco-staging`, and test is `bigtubeco-test`.

Continuing the naming scheme, create a Github repository with a complete installation of Drupal 7 in the root of a repository named `bigtubeco`. The name of the organization or the origination of a specific user do not matter since `deploy-site` treats users, their forks, and organizations the same.

Users are not supposed to login to the Ubuntu server using a shared account; each user should have their own login credentials and can be managed by a service like winbind or centrify.

The default `/var/www` directory is where `deploy-site` will attempt to deploy files and keep them owned by the www-data user. Users should be granted access to read and write to `/var/www` with filesystem-level ACLs. Users who are all members of a developers group could be granted access with the following:

```
setfacl -m g:developers:7 /var/www
setfacl -m d:g:developers:7 /var/www
setfacl -R -m g:developers:7 /var/www
setfacl -R -m d:g:developers:7 /var/www
```

If you have enough people for role separation, developers would not be granted access to login to or deploy code to the live virtual machine.

## Client-side configuration

Users of the `deploy-site` script should modify their local SSH connection settings to allow for forwarded SSH keys. On Mac OSX, edit `/private/etc/ssh_config` and set

```
ForwardAgent yes
```

The ForwardAgent setting allows the SSH connection to the web virtual machine to forward SSH key credentials to the Github server without additional login.

## Special environment notes

Classic enforces SSL between the web and MySQL database server. The `deploy-site` script attempts to patch the settings.php file with information about SSL keys located in `/etc/ssl/mysql/` on each of the live, staging, and test servers.

Drush is also required for all the features of `deploy-site` to work correctly. Use at least a recent Drush 5.x.