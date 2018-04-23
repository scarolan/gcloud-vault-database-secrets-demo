# Simple database secrets engine demo

This tutorial shows you how to build a simple MySQL Vault database secrets engine demo using GCP and your local machine.  You will end up with three terminal sessions open. One on the MySQL server, one for running Vault commands, and a third for running the Vault server.  This basically mimics an application > Vault > database architecture.

## Prerequisites:
* Vault installed on your local machine
* GCP account configured with a project

## Basic Demo - Dynamic Credentials
First run these commands to stand up an Ubuntu 16.04 instance and get connected to it.  The first command creates a new instance, the second command copies our setup script, and the third one forms an SSH tunnel to port 3306 on the remote machine (in addition to getting us connected).

### Step 1: Configure a MySQL instance

```
gcloud compute instances create mysqlvaultdemo \
  --zone us-central1-a \
  --image-family=ubuntu-1604-lts \
  --image-project=ubuntu-os-cloud

gcloud compute scp --zone us-central1-a \
  scripts/install_mysql_ubuntu.sh \
  mysqlvaultdemo:~/

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

vault write database/config/my-mysql-database \
  plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(localhost:3306)/" \
  allowed_roles="my-role" username="vaultadmin" password="vaultpw"

vault write database/roles/my-role \
  db_name=my-mysql-database \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" default_ttl="1h" max_ttl="24h"
```

Your Vault server is now ready.  Test it with the following commands (replace with your lease):

```
vault read database/creds/my-role
vault lease revoke database/creds/my-role/bb52414e-fd9c-dbf5-61ad-d9718c8b2b81
```

### Step 4: Present your demo

You are now ready to present your demo. On the MySQL server you can show a list of users like this:

```
sudo mysql -uroot -pbananas -e 'select user,password from mysql.user;'
```

Now you can demonstrate creation of a user, then revoking the lease and showing the user disappear from the MySQL server. You can stop here or proceed further for a more advanced demo.

## Advanced Demo - A Vault Enabled App and Database

### Step 1: Show off the Vault UI
Log onto http://localhost:8200 in a web browser. Use `password` as the token to log on (you set this above in Step 2).

### Step 2: Create a policy in the UI
Create a policy called `db_read_only` with the following contents:
```
# Allow read only access to employees database
path "database/creds/my-role" {
    capabilities = ["read"]
}
```

### Step 3: Use a periodic token to fetch credentials
Generate a token for your 'app' server:
```
vault token create -period 1h -policy db_read_only
```

Now you can fetch credentials with it:
```
curl -H "X-Vault-Token: 65e8862f-60da-5aee-1d63-7fc360c39132" \
     -X GET http://127.0.0.1:8200/v1/database/creds/my-role | jq .
```

### Step 4: Show renewal of a token using renew-self
```
curl -H "X-Vault-Token: 48422d79-1409-b31b-35d5-17e7b675cf22" \
     -X POST http://127.0.0.1:8200/v1/auth/token/renew-self | jq
```

### Step 5: Show a renewal of a lease associated with credentials
The increment is measured in seconds. Try setting it to 86400 and see what happens when you attempt to exceed the max_ttl.
```
curl -H "X-Vault-Token: 4dcc36f4-f7bf-b19a-df22-f2ad485dd416" \
     -X POST \
     --data '{ "lease_id": "database/creds/my-role/b930c541-6ede-ac4a-2715-c39d9aabcdac", "increment": 3600}' \
     http://127.0.0.1:8200/v1/sys/leases/renew | jq .
```

### Step 6: Log on to MySQL using the dynamic credentials
For this bit you'll need a MySQL client installed on your laptop.  The setup script loads a sample database called employees that you can browse. You will be mimicing the behavior of an application interacting with Vault and a MySQL database.

```
# Use the creds to log on. This is your app connecting to the remote database.
mysql -uv-token-my-role-vz90z2r03tpx4tq5 -pA1a-z2s3r4wzz568y2uw -h 127.0.0.1

# In the SQL prompt run these commands. Your app has read-only access to this database:
use employees;
desc employees;
select emp_no, first_name, last_name, gender from employees limit 10;

# Log off the database server
exit

### Step 7: Revoke the lease
vault lease revoke database/creds/my-role/4f876169-ae19-c69c-34ca-a4ee9e6798d6

### Step 8: Attempt to log on again
mysql -uv-token-my-role-vz90z2r03tpx4tq5 -pA1a-z2s3r4wzz568y2uw -h 127.0.0.1

ERROR 1045 (28000): Access denied for user 'v-token-my-role-vz90z2r03tpx4tq5'@'localhost' (using password: YES)
```

### Optional Stuff

#### Enable audit logs
If you want to show off Vault audit logs just run this command:

```
vault audit enable file file_path=/tmp/my-file.txt
```

### Cleanup
```
gcloud compute instances delete mysqlvaultdemo
```
