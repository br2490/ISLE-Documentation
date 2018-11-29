This section is a checklist of materials to **COPY** from the current running institutional Islandora Production server(s) to the appropriate storage directory on the new ISLE host/server. This guide assumes the remote server has a directory created for your data from production. If you do not have a folder please login to your remote server and create one on a large enough partition to hold your production data.

This is a backup and restore procedure. In short: we are copying the contents of your mysql databases (Drupal and Fedora), webroot, solr data/schema, fedora data, and SSL certificates from an existing Islandora instance to a new host (server). 

**Note**: Ensure you have adequate storage space available for the ISLE host server to accommodate a working copy of a production Islandora's associated configurations and data.

**Note**: _Ubuntu / Debian style paths are used for all examples file locations below, endusers might have different locations for these files HOWEVER the file and directory names etc should be roughly the same._

**Note**: While the ISLE Project recommends use of export methods or tools such as rsync, scp etc., it assumes that endusers are familiar with them and are aware of possible dangers of improperly exporting or copying production data. Ensure adequate backups of any production system(s) are made prior to any attempts. If you are not familiar or are uncomfortable with these processes, it is highly advisable to work with an appropriate IT resource.

---

### Inventory Table
|#| Inventory Item | Setting, Password, or Location | Note |
|-|--|--|--|
|1| Fedora Data Location |  |
|2| Drupal Data Location |  | 
|3| Solr Data Location |  | 
|4| FedoraGSearch Location |  |
|5| MySQL `root` Password |  | If available skip other MySQL users/pass
|6| MySQL Fedora Username |  |
|6| MySQL Fedora Password |  |
|6| MySQL Fedora Database |  |
|7| MySQL Drupal Username |  |
|7| MySQL Drupal Password |  |
|7| MySQL Drupal Database |  |

---

## Taking inventory of your existing Islandora Stack
This section presents common locations of files, and `terminal` commands to find the data required.
To test the common locations, while logged into your server, type `cd {LOCATION} && ls`. If you can change into that directory _record that location_, otherwise use the `command` provided and record the output.

### Locating your Fedora, Drupal (Islandora), and Solr data folders

 0. Login to your existing Islandora Server. 

 1. Finding your Fedora data folder
    - Common locations: `/usr/local/fedora/data` or `/usr/local/tomcat/fedora/data`  
    - Use find: `find / -type d -ipath '*fedora/data' -ls  2>/dev/null`
             
 2. Finding your Drupal data folder  
    - Common location: `/var/www/` (likely in a sub-folder; e.g., html, islandora, etc.)
    - `grep --include=index.php -rl -e 'Drupal' / 2>/dev/null`

 3. Finding your Solr data folder  
    - Common location: `/usr/local/solr`, `/usr/local/tomcat/solr`, or `/usr/local/fedora/solr`
    - `find / -type d -ipath '*solr/*/data' -ls  2>/dev/null`

 4. Finding your FedoraGSearch data (i.e. transforms) folder
    - `find / -type d -ipath '*web-inf/classes/fgsconfigfinal' -ls 2>/dev/null`

### MySQL data for both Fedora and Drupal _OR_ your MySQL root password.

 5. Note well that if you have your MySQL `root` password you only need find the database and host values below.

 6. Finding your Fedora MySQL username, password, and database
    - `grep --include=fedora.fcfg -rnw -e 'name="dbUsername"' -e 'name="dbPassword"' -e 'name="jdbcURL"' / 2>/dev/null`
    - This command _will_ print multiple lines. The first three lines are important but please save the rest (just in case).

    Example output:
     > param name="dbUsername" value="**fedoraDB**"  
     > param name="jdbcURL" value="jdbc:mysql://localhost/**fedora3**?useUnicode=true&amp;amp;characterEncoding=UTF-8&amp;amp;autoReconnect=true"  
     > param name="dbPassword" value="**zMgBM6hGwjCeEuPD**"

    - Username: Copy the value from `dbUsername value=`
    - Password: Copy the value from `dbPassword value=`
    - Database: Copy from the value `jdbcURL value=` the database name which is directly between the "/" and the only "?"
    - Hostname: If other than _localhost_ please take note of the hostname.

 7. Finding your Drupal MySQL username, password, and database
    - `grep --include=filter-drupal.xml -rnw -e 'dbname.*user.*password.*"' / 2>/dev/null`
    
    Example output:
      > connection server="localhost" port="3306" dbname="**islandora**" user="**drupalIslandora**" password="**Kjs8n5zQXfPNhZ9k**"

    - Username: copy the value from `user=`
    - Password: copy the value from `password=`
    - Database: copy the value from `dbname=`
    - Hostname: If other than _localhost_ please take note of the hostname.


## Backing up the _required_ files from your existing Islandora Stack.

 0. Login to your existing Islandora Server. 
    - We will be create backups in your home directory to find them easily; 
    - Change directory to your home directory with `cd ~`. 
    - Create a new directory in your home folder: `mkdir isledata && cd isledata` before starting.
      - Your backups will be stored in `~/isledata` (`cd ~/isledata` to return to this location)
    - If you will be copying data to a remote server please have SSH access to that server.
      - Test your connection to the remote server before starting: `ssh {USER}@{SERVER}` 

 1. Generate SQL Dumps 
    > **Note**  you may add your password directly to the commands below as: `-p{PASSWORD}` (no additional space).
    This is **not** recommended as your shell history (e.g., `bash_history`) will have those passwords stored. You may delete your shell history when you are complete (`rm ~\.bash_history`)

    **Note** If you have your MySQL root password replace {USERNAMES} with `root`
     - Make to be in our backup location `cd ~/isledata`
     - SQL dump of your Fedora database
        - `mysqldump -u {FEDORA_USERNAME} -p {FEDORA_DATABASE_NAME} | gzip > fedora.sql.gz`
     - SQL dump of your Drupal database
        - `mysqldump -u {DRUPAL_USERNAME} -p {DRUPAL_DATABASE_NAME} | gzip > drupal.sql.gz`

#### MySQL Notes

* _Drupal website databases can have a multitude of names and conventions. Confer with the appropriate IT resources for your institution's database naming conventions._

* _CLI == Command line_

* _Recommended that the production databases be exported using the `.sql` and `.gz` file formats e.g. `drupal_site_2018.sql.gz` for better compression and minimal storage footprint._

* _If the enduser is running multi-sites, there will be additional databases to export._

* _Do not export the `fedora3` database as it will be recreated by the SQL index in later steps of the Migration Guide_

* _If possible, on the production Apache webserver, run `drush cc all` from the command line on the production server in the appropriate sites directory PRIOR to db export(s)_
  * _Otherwise issues can occur on import due to all cache tables being larger than `innodb_log_file_size` allows_

#### MySQL Tools for Export
Here are a few pieces of documentation specific for the tasks above.

**Caution**: While the ISLE Project recommends use of export methods or tools such as mysqldump etc., it assumes that endusers are familiar with them and are aware of possible dangers of improperly exporting or copying production data. Ensure adequate backups of any production system(s) are made prior to any attempts. If you are not familiar or are uncomfortable with these processes, it is highly advisable to work with an appropriate IT resource.

* [Official MySQL documentation](https://dev.mysql.com/doc/)
    * [mysqldump console utility documentation](https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html)
    * [Digital Ocean quick start for mysqldump export](https://www.digitalocean.com/community/tutorials/how-to-import-and-export-databases-in-mysql-or-mariadb)
* [Official MySQL GUI app - Workbench](https://www.mysql.com/products/workbench/) _For Linux, MacOS and Windows_
* [Sequel Pro](https://sequelpro.com/) _MacOS only_


 2. Drupal (Islandora) webroot
      - In our backup location `cd ~/isledata`
      - `tar -zcf drupal-web.tar.gz -C {DRUPAL_DATA_LOCATION} .` (don't forget the final `.`)
        - for example: `tar -zcf drupal-web.tar.gz -C /var/www/html .`
      - Permission error? Prepend the command with `sudo` (e.g., `sudo tar -zcf drupal-web.tar.gz -C /var/www/html .`)

* Copy the following below from the Islandora Production Server(s) to the suggested directories `/current_prod_islandora_config/apache/` or `/data/apache/` on the ISLE directories located on both the local ISLE config laptop and remote ISLE Host server as designated.

    * `/current_prod_islandora_config/apache/` should be a directory used for configurations to be merged or edited on an enduser's laptop (`local ISLE config laptop`)

    * `/data/apache/`should be a directory used for apache website code and data  to be copied to a storage area on the target ISLE Host server (`Remote ISLE Host server`)  due to size.

This data and these configurations will be used in conjunction with an Apache container.

| Data          | Description                 | Possible Location             | Suggested ISLE Path Destination        | Copy location |
| ------------- | -------------               | -------------                 | -------------                          | ------------- |
| html          | Islandora/Drupal Website    | /var/www/                     | yourdomain-data/apache/html/           | Remote ISLE Host server  |
| yoursite.conf | Apache webserver vhost file | /etc/apache2/sites-enabled/   | /current_prod_islandora_config/apache/ | Local ISLE config laptop |


#### Apache Notes

* /var/www/`html`

    * _Entire contents unless size prohibits should be copied to a directory on the remote ISLE host server._

    * _If `html` is not used, then substitute with the appropriate directory for the Islandora / Drupal site_

* `yoursite.conf`

    * _File will have different name but this should be the enabled Apache vhost file of your production Islandora website._
    * _There may also be a seperate vhost that uses SSL and https. Copy that too if available._


 3. Solr Data
      - In our backup location `cd ~/isledata`
      - `tar -zcf solr.tar.gz -C {SOLR_DATA_LOCATION} .`
        - for example: `tar -zcf solr-data.tar.gz -C /usr/local/solr .`
      - Permission error? Prepend the command with `sudo` (e.g., `sudo tar -zcf solr-data.tar.gz -C /usr/local/solr .`)

Copy the following below from the Islandora Production Server(s) to the suggested directory to the ISLE directory `current_prod_islandora_config/solr` located on the local ISLE config laptop.

This data will be used in conjunction with a SOLR container.

| Data           | Description                              | Possible Location                                             | Suggested ISLE Path Destination     | Copy location |
| -------------  | -------------                            | -------------                                                 | -------------                       | ------------- |
| schema.xml     | Solr schema file                         | /var/lib/tomcat7/webapps/solr/collection1/conf/schema.xml     | /current_prod_islandora_config/solr | Local ISLE config laptop |
| solrconfig.xml | Solr config file                         | /var/lib/tomcat7/webapps/solr/collection1/conf/solrconfig.xml | /current_prod_islandora_config/solr | Local ISLE config laptop |
| stopwords.txt  | Solr file for filtering out common words | /var/lib/tomcat7/webapps/solr/collection1/conf/stopwords.txt  | /current_prod_islandora_config/solr | Local ISLE config laptop |


 4. Fedora Generic Search (FGS) Transforms (_optional_)
      - In our backup location `cd ~/isledata`
      - `tar -zcf fgs.tar.gz -C {FEDORAGSEARCH_LOCATION} .`
        - for example: `tar -zcf fgs.tar.gz -C /tomcat/webapps/fedoragsearch/WEB-INF/classes/fgsconfigFinal .`
      - Permission error? Prepend the command with `sudo` (e.g., `sudo tar -zcf fgs.tar.gz -C /tomcat/webapps/fedoragsearch/WEB-INF/classes/fgsconfigFinal .`)

Copy the following below from the Islandora Production Server(s) to the suggested directory `current_prod_islandora_config/gsearch/` on the ISLE directory located on the local ISLE config laptop.

This data will be used in conjunction with a Fedora container.

| File / Directory | Description   | Possible Location   | Suggested ISLE Path Destination         | Copy location |
| -------------    | ------------- | -------------       | -------------                           | ------------- |
| islandora_transforms | Transformation XSLTs directory | /var/lib/tomcat7/webapps/fedoragsearch/WEB-INF/classes/fgsconfigFinal/index/FgsIndex/ | /current_prod_islandora_config/gsearch/ | Local ISLE config laptop |
| foxmlToSolr.xslt | "top-level" transformational XSLT | /var/lib/tomcat7/webapps/fedoragsearch/WEB-INF/classes/fgsconfigFinal/index/FgsIndex/ | /current_prod_islandora_config/gsearch/ | Local ISLE config laptop |


 5. Fedora Data  
    > These are large directories and copying them _will_ take several hours or even days. Please plan accordingly and prepare to leave these processes running unattended.

    The following folders are all located in the recorded {FEDORA_DATA_LOCATION}
    - These folders will be copied _directly_ to the remote server or to a new directory due to their size:
      - Fedora datastreamStore
      - Fedora objectStore
      - Fedora resourceIndex

Copy the following below from the Islandora Production Server(s) to the suggested directory `current_prod_islandora_config/fedora/` on the ISLE directories located on both the local ISLE config laptop and remote ISLE Host server as designated.


This data will be used in conjunction with a Fedora container.  

| File / Directory      | Description                   | Possible Location                | Suggested ISLE Path Destination                   | Copy location            |
| -------------         | -------------                 | -------------                    | -------------                                     | -------------            |
| datastreamStore       | Entire Fedora data directory  | /usr/local/fedora/data/          | yourdomain-data/fedora/data/datastreamStore       | Remote ISLE Host server  |
| fedora-xacml-policies | Entire Fedora data directory  | /usr/local/fedora/data/          | yourdomain-data/fedora/data/fedora-xacml-policies | Remote ISLE Host server  |
| objectStore           | Entire Fedora data directory  | /usr/local/fedora/data/          | yourdomain-data/fedora/data/objectStore           | Remote ISLE Host server  |


#### Fedora Notes

* Do not copy the following directories from the production Islandora fedora `/usr/local/fedora/data` directory.
        * /usr/local/fedora/`data/activemq-data`
        * /usr/local/fedora/`data/resourceIndex`
