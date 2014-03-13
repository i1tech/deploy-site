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

Each deploy also creates a database backup. The backups are stored as `/var/dbbackup/${ENVIRON}-{TAG}.sql` using Drush by default.

## Server-side configuration

Running `deploy-site` is automatically setup to use the hostname of the Ubuntu server to locate the matching repository on Github. Hostnames and repositories are standardized by customer.

Take the Big Tube Company as an example, a fictional company. They host their website at http://bigtubeco.com. Their staging and test site FQDN are staging.bigtubeco.com and test.bigtubeco.com. The hostname of their production Ubuntu VM would be named `bigtubeco`, the staging server hostname `bigtubeco-staging`, and test is `bigtubeco-test`.

Continuing the naming scheme, create a Github repository with a complete installation of Drupal 7 in the root of a repository named `bigtubeco`. The name of the organization or the origination of a specific user does not matter since `deploy-site` treats users, their forks, and organizations the same.

Users are not supposed to login to the Ubuntu server using a shared account; each user should have their own login credentials and can be managed by a service like winbind, Centrify or Likewise.

The default `/var/www` directory is where `deploy-site` will attempt to deploy files and set appropriate owner, group, filesystem permissions and acl's. In our environment, this means owner/group would be www-data, and then we would apply ACL's to allow Developers, Domain Admins and any other AD Group users as required. Users should be granted access to read and write to `/var/www` with filesystem-level ACLs. Example, users who are members of a developers group could be granted access with the following:

```
setfacl -R -m g:developers:rwx,d:g:developers:rwx /var/www
```

If you have enough people for role separation, developers would not be granted access to log on or deploy code to the live systems.

**UPDATE: 04-09-2013**

Recent editions to the deploy site script require a few changes to Apache to fully reap the benefits of the additions, and should increase Apache's performance.

First you will need to modify the default site file (assuming your setup is 1 web site per VM).  Change all occurences of "AllowOverride ??????" to be "AllowOverride None".  Next, below the <Directory /var/www> ... </Directory> configuration block, add the line "Include htaccess.d/".  What you should end up with is something like this:

```
    DocumentRoot /var/www
    <Directory />
      Options FollowSymLinks
      AllowOverride None
    </Directory>
    <Directory /var/www/>
      Options Indexes FollowSymLinks MultiViews
      AllowOverride None
      Order allow,deny
      allow from all
      <Files ~ "^install.php">
        Order deny,allow
        Deny from all
      </Files>
    </Directory>

    Include htaccess.d/
```

**__NOTE__**:  The above is only a portion of the actual configuration file.

In order to implement the above changes, Apache MUST be restarted.  However, if we do this now, our Drupal installation will be broken, as NO .htaccess files will be loaded on access to the site. If we now run the deploy-site script, it will finish up appropriate configuration by adding Includes for each .htaccess file it finds during the deploy to the /etc/apache2/htaccess.d/htaccess.conf file. It will also take care of restarting apache after the new configuration file has been generated.

A new --no-backup switch has also been added.  This switch does exactly what it implies, NO database backup will be created prior to deploying the new code.  We DO NOT recommend using this switch on your production server!  Switch was implemented to shorten the deployment times developers were seeing when deploying test branches of the code, in this situation backups might not be required, or make sense.

A new variable was added to allow special/non-standard ports to be used for your github host.  When setting this variable, be sure to include the colon at the beginning (ex: PORT=":7999") would be used for the default configuration if you are using Atlassian's Stash product as your git repository server. 

## Client-side configuration

Users of the `deploy-site` script should modify their local SSH connection settings to allow for forwarded SSH keys. On Mac OSX, edit `/private/etc/ssh_config` and set

```
ForwardAgent yes
```

The ForwardAgent setting allows the SSH connection to the web virtual machine to forward SSH key credentials to the Github server without additional login.

## Special environment notes

Classic enforces SSL between the web and MySQL database server. The `deploy-site` script attempts to patch the settings.php file with information about SSL keys located in `/etc/ssl/mysql/` on each of the live, staging, and test servers.

Drush is also required for all the features of `deploy-site` to work correctly. Use at least a recent Drush 5.x.