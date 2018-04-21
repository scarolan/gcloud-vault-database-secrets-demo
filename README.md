# Simple database secrets engine demo

This tutorial shows you how to build a simple MySQL Vault database secrets engine demo using GCP and your local machine.  You will end up with three terminal sessions open. One on the MySQL server, one for running Vault commands, and a third for running the Vault server.  This basically mimics an application > Vault > database architecture.

## Prerequisites:
* Vault installed on your local machine
* GCP account configured with a project

## Instructions:
First run these commands to stand up an Ubuntu 16.04 instance and get connected to it.  The first command creates a new instance, the second command copies our setup script, and the third one forms an SSH tunnel to port 3306 on the remote machine (in addition to getting us connected).

### Step 1: Configure a MySQL instance

```
gcloud compute instances create mysqlvaultdemo --zone us-central1-a --image-family=ubuntu-1604-lts --image-project=ubuntu-os-cloud
gcloud compute scp --zone us-central1-a scripts/install_mysql_ubuntu.sh mysqlvaultdemo:~/
gcloud compute ssh --zone us-central1-a mysqlvaultdemo -- -L 3306:localhost:3306
```

Now on the gcloud instance you just ssh'd into, run this:

```
./install_mysql_ubuntu.sh
```

The MySQL server is now ready, and port 3306 is forwarded back to your machine.  

### Step 2: Configure Vault on your local machine

Run the following commands to get Vault setup on your laptop:

```
vault server -dev -dev-root-token-id=password
```

Leave Vault running in this terminal.  

### Step 3: Configure the Vault database backend for MySQL

Open another terminal and run these:

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

### Step 4: Present your demo

You are now ready to present your demo. On the MySQL server you can show a list of users like this:

```
sudo mysql -uroot -pbananas
```

Once inside the MySQL command prompt:

```
select user,password from mysql.user;
```

Now you can demonstrate creation of a user, then revoking the lease and showing the user disappear from the MySQL server.

### Optional Extras
#### Log on using the dynamic credentials
For this bit you'll need a mysql client installed on your laptop.  The setup script loads a sample database called employees that you can browse.

```
# Grab some creds if you haven't already
vault read database/creds/my-role
# Or if you want to get fancy, use the API.  (jq will make the output pretty)
curl -H "X-Vault-Token: password" -X GET http://127.0.0.1:8200/v1/database/creds/my-role | jq

# Use the creds to log on
mysql -uv-token-my-role-vz90z2r03tpx4tq5 -pA1a-z2s3r4wzz568y2uw -h 127.0.0.1

# In the SQL prompt run these commands:
use employees;
desc employees;
select emp_no, first_name, last_name, gender from employees limit 10;

# Log off the machine, revoke the lease.  Attempt to log on again:
vault lease revoke database/creds/my-role/4f876169-ae19-c69c-34ca-a4ee9e6798d6
mysql -uv-token-my-role-vz90z2r03tpx4tq5 -pA1a-z2s3r4wzz568y2uw -h 127.0.0.1

ERROR 1045 (28000): Access denied for user 'v-token-my-role-vz90z2r03tpx4tq5'@'localhost' (using password: YES)
```

#### Use a periodic token to fetch credentials
Create a policy like this:
```
# Allow read only access to employees database
path "database/creds/my-role" {
    capabilities = ["read"]
}
```

Generate a token for your 'app' server:
```
vault token create -period 1h -policy db_read_only
```

Now you can fetch credentials with it:
```
curl -H "X-Vault-Token: 65e8862f-60da-5aee-1d63-7fc360c39132" -X GET http://127.0.0.1:8200/v1/database/creds/my-role | jq .
```

#### Enable audit logs
If you want to show off Vault audit logs just run this command:

```
vault audit enable file file_path=/tmp/my-file.txt
```

### Cleanup
```
gcloud compute instances delete mysqlvaultdemo
```
