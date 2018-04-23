# Simple database secrets engine demo

This tutorial shows you how to build a simple MySQL Vault database secrets engine demo using GCP and your local machine.  You will end up with three terminal sessions open. One on the MySQL server, one for running Vault commands, and a third for running the Vault server.  This basically mimics an application > Vault > database architecture.

## Prerequisites:
* Vault installed on your local machine
* GCP account configured with a project

## Basic Demo - Dynamic Credentials
First run these commands to stand up an Ubuntu 16.04 instance and get connected to it.  The first command creates a new instance, the second command copies our setup script, and the third one forms an SSH tunnel to port 3306 on the remote machine (in addition to getting us connected).

### Step 1: Configure a MySQL instance
Open a terminal window and run the following commands:

```
gcloud compute instances create mysqlvaultdemo \
  --zone us-central1-a \
  --image-family=ubuntu-1604-lts \
  --image-project=ubuntu-os-cloud
  
# You may have to wait a few seconds before running the next command. GCP is fast, but not *that* fast.

gcloud compute scp --zone us-central1-a \
  scripts/install_mysql_ubuntu.sh \
  mysqlvaultdemo:~/

gcloud compute ssh --zone us-central1-a mysqlvaultdemo -- -L 3306:localhost:3306
```

Now on the gcloud instance you just ssh'd into, run this:

```
./install_mysql_ubuntu.sh
```

The MySQL server is now ready, and port 3306 is forwarded back to your machine. Do not proceed to the next steps until the script has completely finished running.

### Step 2: Configure Vault on your local machine
In a new terminal, run the following commands to get Vault setup on your laptop:

```
vault server -dev -dev-root-token-id=password
```

Leave Vault running in this terminal. You can point out API actions as they are logged, such as revoked leases, etc.

### Step 3: Configure the Vault database backend for MySQL
Open a third terminal and run these commands.  You can copy and paste all of this code in one block:

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

### Step 3: Generate a periodic token for your 'app'
Generate a token for your 'app' server.  Export it into a variable for ease of use. Replace with *your* token. 
```
vault token create -period 1h -policy db_read_only
export APP_TOKEN=47b422f4-ddeb-671a-756a-c161b66a84dd
```

Now you can fetch credentials with it:
```
curl -H "X-Vault-Token: $APP_TOKEN" \
     -X GET http://127.0.0.1:8200/v1/database/creds/my-role | jq .
```

### Step 4: Log on to MySQL using the dynamic credentials
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
```

### Step 5: Revoke the lease
Use the lease id that you generated in step 3. Scroll back in your terminal to find it.
```
vault lease revoke database/creds/my-role/f9cf14cd-806c-50c4-8738-0232129bdd0b

# You can also revoke *all* the leases with this prefix:
vault lease revoke -prefix database/creds/my-role
```

### Step 6: Attempt to log on again
```
mysql -uv-token-my-role-vz90z2r03tpx4tq5 -pA1a-z2s3r4wzz568y2uw -h 127.0.0.1

ERROR 1045 (28000): Access denied for user 'v-token-my-role-vz90z2r03tpx4tq5'@'localhost' (using password: YES)
```

### Optional Stuff

#### Show renewal of a token using renew-self
```
curl -H "X-Vault-Token: $APP_TOKEN" \
     -X POST http://127.0.0.1:8200/v1/auth/token/renew-self | jq
```

#### Show a list of all leases
You can do this in the UI or with this command:
```
vault list /sys/leases/lookup/database/creds/my-role/
```

Export the lease id that you are using into a variable. You'll use this in the next step.
```
export MY_LEASE=a8174038-a3e5-7495-170f-6ab92ece4c22
```

#### Show a renewal of a lease associated with credentials
The increment is measured in seconds. Try setting it to 86400 and see what happens when you attempt to exceed the max_ttl.
```
curl -H "X-Vault-Token: $APP_TOKEN" \
     -X POST \
     --data '{ "lease_id": "database/creds/my-role/$MY_LEASE", "increment": 3600}' \
     http://127.0.0.1:8200/v1/sys/leases/renew | jq .
```

#### Revoke all leases and invalidate all active credentials
```
# Run this five or six times to generate credentials
vault read database/creds/my-role
# Show all the users on the database server
sudo mysql -uroot -pbananas -e 'select user,password from mysql.user;'
# Revoke everything under my-role
vault lease revoke -prefix database/creds/my-role

#### Enable audit logs
If you want to show off Vault audit logs just run this command:

```
vault audit enable file file_path=/tmp/my-file.txt
```

### Cleanup
```
gcloud compute instances delete mysqlvaultdemo
```
