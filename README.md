# Simple database secrets engine demo

This tutorial shows you how to build a simple MySQL Vault database secrets engine demo using GCP and your local machine.  You will end up with three terminal sessions open. One on the MySQL server, one for running Vault commands, and a third for running the Vault server.  This basically mimics an application > Vault > database architecture.

## Prerequisites:
* Vault installed on your local machine
* GCP account configured with a project

## Instructions:
First run these commands to stand up an Ubuntu 16.04 instance and get connected to it.  The first command creates a new instance, the second command copies our setup script, and the third one forms an SSH tunnel to port 3306 on the remote machine (in addition to getting us connected).

```
gcloud compute instances create mysqlvaultdemo --zone us-central1-a --image-family=ubuntu-1604-lts --image-project=ubuntu-os-cloud
gcloud compute scp --zone us-central1-a scripts/install_mysql_ubuntu.sh mysqlvaultdemo:~/
gcloud compute ssh --zone us-central1-a mysqlvaultdemo -- -L 3306:localhost:3306
```

Now on the gcloud instance you just ssh'd into, run this:

```
./install_mysql_ubuntu.sh
```

The MySQL server is now ready, and port 3306 is forwarded back to your machine.  Run the following commands to get Vault setup on your laptop:

```
vault server -dev -dev-root-token-id=password
```

Leave Vault running in this terminal.  Open another terminal and run these:

```
export VAULT_ADDR=http://127.0.0.1:8200

vault secrets enable database

vault write database/config/my-mysql-database plugin_name=mysql-database-plugin connection_url="{{username}}:{{password}}@tcp(localhost:3306)/" allowed_roles="my-role" username="vaultadmin" password="vaultpw"

vault write database/roles/my-role db_name=my-mysql-database creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" default_ttl="1h" max_ttl="24h"
```

Your Vault server is now ready.  Test it with the following commands (replace with your lease):

```
vault read database/creds/my-role
vault lease revoke database/creds/my-role/bb52414e-fd9c-dbf5-61ad-d9718c8b2b81
```

You are now ready to present your demo. On the MySQL server you can show a list of users like this:

```
sudo mysql -uroot -pbananas
```

Once inside the MySQL command prompt:

```
select user,password from mysql.user;
```

Now you can demonstrate creation of a user, then revoking the lease and showing the user disappear from the MySQL server.

#### Optional - Enable audit logs
If you want to show off Vault audit logs just run this command:

```
vault audit enable file file_path=/tmp/my-file.txt
```

### Cleanup
```
gcloud compute instances delete mysqlvaultdemo
```
