# Docker Mediawiki installation on ARM

## Install from scratch

Follow those simple steps: 

1. Clone the git repo
    ```bash
    git clone https://github.com/patklaey/docker-mediawiki
    cd docker-mediawiki
    ```
1. Copy the env template to have your own version of it
    ```bash
    cp .env-template .env
    ```
1. Modify the ```.env``` file to change passwords
    ```bash
    vi .env
    ```
1. Start up the containers
    ```bash
    docker-compose up -d
    ```
1. Point your browser to your wiki, you should now see the Mediawiki installation wizard (remember it's on
 port 8001). So for example: http://192.168.1.2:8001/
    * For "Database host" use "database"
    * For "Database name (no hyphens)", "Database username" and "Database password" use the same values as you've set 
    in ```.env```
1. Download the ```LocalSettings.php``` file and put it into the current folder (where also you have the 
docker-compose.yml file)
1. Stop the docker containers
    ```bash
    docker-compose stop
    ```
1. Edit the docker-compose.yml
    ```bash
    vi docker-compose.yml
    ```
1. Uncomment the line saying ```- ./LocalSettings.php:/var/www/html/LocalSettings.php``` to copy the just generated 
LocalSettings.php file to your web container on startup
1. Optionally install extensions (see [Install an extension](#install-an-extension)) -> without restarting the web 
container)
    * This installation contains the following extensions already
        * SyntaxHighlight
        * MobileFrontend
        * CreatePage
    * You might want to add the following lines to your ```LocalSettings.php``` file
    ```php
    require_once "$IP/extensions/SyntaxHighlight_GeSHi/SyntaxHighlight_GeSHi.php";
    wfLoadExtension( 'MobileFrontend' );
    $wgMFDefaultSkinClass = 'SkinVector';
    require_once "$IP/extensions/CreatePage/CreatePage.php";
    ```
1. Start the containers again
    ```bash
    docker-compose up -d
    ```
1. Point your browser again to your wiki and you should be good to go

## Install from existing wiki

1. Clone the git repo
    ```bash
    git clone https://github.com/patklaey/docker-mediawiki
    cd docker-mediawiki
    ```
1. Copy the env template to have your own version of it
    ```bash
    cp .env-template .env
    ```
1. Modify the ```.env``` file to change passwords
    ```bash
    vi .env
    ```
1. Download the ```LocalSettings.php``` file from your existing installation to the current folder
1. Uncomment the line saying ```- ./LocalSettings.php:/var/www/html/LocalSettings.php``` to copy the just downloaded
LocalSettings.php file to your web container on startup 
1. Edit the ```LocalSettings.php``` file to match the new environment
    ```bash
    vi LocalSettings.php
    ```
    * Change ```$wgDBserver = "your-old-endpoint";``` to ```$wgDBserver = "database";```
    * Comment/Remove all installed plugins ```require_once "$IP/...";``` or ```wfLoadExtension( '...' );``` or install
    them again (see [Install an extension](#install-an-extension))
        * This installation contains the following extensions already
            * SyntaxHighlight
            * MobileFrontend
            * CreatePage
        * You might want to add the following lines to your ```LocalSettings.php``` file
        ```php
        require_once "$IP/extensions/SyntaxHighlight_GeSHi/SyntaxHighlight_GeSHi.php";
        wfLoadExtension( 'MobileFrontend' );
        $wgMFDefaultSkinClass = 'SkinVector';
        require_once "$IP/extensions/CreatePage/CreatePage.php";
        ```
    * Make sure to remove unsupported/outdated skins ``wfLoadSkin( '...' );``
1. Start the containers again
    ```bash
    docker-compose up -d
    ```
1. Copy the database backup into the mysql container
    ```bash
    docker cp /path/to/backup/wiki-utf.sql mediawiki-db:/root
    ```
1. Import the database backup
    ```bash
    source .env
    docker exec mediawiki-db sh -c "mysql --default-character-set=latin1 -u ${DB_USERNAME} --password=${DB_PASSWORD} ${DB_NAME} < /root/wiki-utf.sql"  
    ```
1. Point your browser to your %wiki%/mw-config/, you should now see the Mediawiki installation wizard (remember it's on
 port 8001). So for example: http://192.168.1.2:8001/mw-config/
    * Get the upgrade key from your ```LocalSettings.php``` file (```$wgUpgradeKey```)
    * Upgrade the DB schema

Congrats, you're done!

### Troubleshooting

* Should you see the following error
    ```
    [41e5e01ec436cab12d3313d8] /mw-config/?page=ExistingWiki MWException from line 187 of /var/www/html/includes/MagicWord.php: Error: invalid magic word 'createpage'
    ```
    The upgrade cannot process the CreatePage extension. Therefore simply comment the line
    ```php
    require_once "$IP/extensions/CreatePage/CreatePage.php";
    ```
    in the ```LocalSettings.php``` file, restart the mediawiki-web container (```docker restart mediawiki-web```) and 
    restart the upgrade. When the upgrade is done, uncomment the line again, restart the container and everything should
    be good-

## Install an extension

The following steps assume that you have already installed mediawiki according to the guides above

1. Go to the [MediaWiki Extensions Page](https://www.mediawiki.org/wiki/Special:ExtensionDistributor) and search your
extension
1. Download and extract the extension to the extensions directory (follow basically the installation steps of the 
extension)
    ```bash
    cd extensions
    wget %extension-url%
    tar -xzf %extension%.tar.gz
    rm %extension%.tar.gz
    cd ..
    ```
1. Modify the ```LocalSettings.php``` file according to the extensions install sections
    ```bash
    vi LocalSettings.php
    ```
1. Restart the mediawiki web container to load the extension
    ```bash
    docker restart mediawiki-web
    ```
## Upgrade
1. Backup the DB
    ```bash
    source .env
    docker exec mediawiki-db sh -c "mysqldump -u ${DB_USERNAME} --password=${DB_PASSWORD} --opt --quote-names --skip-set-charset --default-character-set=latin1 ${DB_NAME} > /backup/wiki-utf-pre-upgrade.sql"
    ```
1. Change the version in the docker-compose.yml
    ```bash
    vi docker-compose.yml
    ```
1. Stop the service
    ```bash
    docker-compose down
    ```
1. Edit the LocalSettings.php file and comment the createPage widget (otherwise upgrade will fail)_ 
    ```bash
    vi LocalSettings.php
    ```
    ```php
    #require_once "$IP/extensions/CreatePage/CreatePage.php";
    ```
1. Start the service again
    ```bash
    docker-compose up -d && docker-compose logs -f
    ```
1. Point the browser to %wiki-url%/mw-config to perform the DB upgrade (get the upgrade key from you LocalSettings.php)
1. If everything is done, uncomment the CreatePage extension again and restart the services
     ```bash
     vi LocalSettings.php
     ```
     ```php
     require_once "$IP/extensions/CreatePage/CreatePage.php";
     ```
     ```bash
     docker-compose down
     docker-compose up -d
     ```