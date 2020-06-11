# Hashicorp Vault HA Deployment in Azure

This repo contains all the code required to deploy & configure Hashicorp Vault on Azure.


## Key features

* HA Vault deployed in AKS
* A managed Azure MySQL Database configured as the storage backend.
* An Azure KeyVault server used for auto unsealing Vault and storing Vault's Root Token & Recovery Keys.
* User authentication through Azure AD

![Overall architecture](diagram/vault-diagram.png)

_Diagram can be updated through `diagram/vault-diagram.xml` with draw.io_

## CI/CD
The infra-vault is using Argo for CI at the moment.
It is currently only the terraform plan step that is ran when pushing to the repository. 
The build results can be found at https://argo-platform.maersk-digital.net

TODO: At a later point we would like to have a full CI/CD pipeline using Argo, but for now, please follow the steps under `Deploying Vault`.

## Deploying Vault

### Pre-requisites

1. terraform

1. helm

1. az (Azure CLI)

1. kubectl

1. vault

### Deploying

1. Clone the [maersk-analytics/infra-vault](https://bitbucket.org/maersk-analytics/infra-vault/src/master) repo and cd into the root of the repo.

1. For new deployments, create a new environment with the `var.<ENVIRONMENT_NAME>.secret.mk` file for your deployment based on the `vars.example.secret.mk` and populate the values as required.

1. To deploy Hashicorp Vault use the following command. (Make sure your replace the <ENVIRONMENT_NAME> with your value):

```shell
make apply -e REMOTE_ENV="<ENVIRONMENT_NAME>"
```

## Destroying a Vault Deployment

Note the following steps will delete a Vault Server, including all other services deployed as part of the pipeline (ie. MySQL server & AZ KeyVault).

### Steps to Destroy a Vault Server deployed via Drone

1. Switch to the branch of the vault environment you wish to destroy.

1. To destroy the environment create a new commit with message "DESTROY" and push to the remote branch. This will trigger the Vault deletion pipeline in Drone.

```shell
git commit --allow-empty -m "DESTROY"
git push origin <YOUR_BRANCH>
```

### Steps to Destroy a Vault Server deployed via your Local Machine

To destroy Hashicorp Vault use the following command. (Make sure your replace the <ENVIRONMENT_NAME> with your value.):

```shell
make destroy -e REMOTE_ENV="<ENVIRONMENT_NAME>"
```

## Protect vault against accidental deletion

Azure allows resources to be locked with "CanNotDelete" option. The resource can not be deleted unless the lock is removed.

Note: Refer [Azure documentation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-lock-resources) - To create or delete management locks, you must have access to `Microsoft.Authorization/*` or `Microsoft.Authorization/locks/*` actions. Of the built-in roles, only `Owner` and `User Access Administrator` are granted those actions.

This can not run in the CI/CD pipeline as the Service Principal do not have the requisite privileges. Run this from the local machine - additional tools required: *jq, xargs, sed and bash*.

Use "az login" to login as Admin : `digital-platform-admins` can use respective ADM account and then proceed with the following.

### Steps to apply locks

Switch to the branch of the vault environment you wish to lock against deletion.

```shell
make lock_it -e REMOTE_ENV="<ENVIRONMENT_NAME>"
```

### Steps to remove the locks

To remove the locks on resources.

```shell
make unlock -e REMOTE_ENV="<ENVIRONMENT_NAME>"
```

This will remove ONLY those locks which had been created using this script.

### Steps to list the locks

To list the locks

```shell
make lock_info -e REMOTE_ENV="<ENVIRONMENT_NAME>"
```

This will list ALL the locks in the resource-group, including those which may not have been created by this script.

## User Authentication

The Vault deployment is configured to use the Vault OIDC auth method with Azure AD for user authentication.

The Azure App Registration is configured, such that only members of the AD group `digital-vault-users` can login to Vault.

Note, that reply URLs is not automatically added to the app registration, so this may produce errors with non-standard deployments.

By default, the AD group `digital-platform-admins` are added as an "admin" with the `policy-helper/policies/admin-policy.hcl`.

## Users, Groups and Policies

Users (in Vault, this is called `entities`) are automatically added at login, and user information (name, email) are stored in `entity alias` metadata.

Groups, Polices, Secret Engines and other Vault structures are currently managed in another repository, [maersk-analytics/vault-operations](https://bitbucket.org/maersk-analytics/vault-operations/src/master/).

This repo is _linked_ by the `app role` created in `auth-user-helper/add-vault-ops-runner.sh` and the variable `VAULT_OPS_RUNNER_SECRET` must be shared per environment in both repos. Currently, environments are mapped as:

```text
<infra-vault environment> <> <vault-operations environment>
develop <> dev
master <> prod
```

## SSL enforcement for MySQL
There is enabled SSL for the connection between Vault and the MySQL backend/DB. Using the instructions described at https://docs.microsoft.com/en-us/azure/mysql/howto-configure-ssl

The certificate `BaltimoreCyberTrustRoot.crt.pem` has been added to the repository. The certificate is added as a secret in the Kubernetes namespace and later mounted into the Vault container for usage.

SSL enforcement is enabled on the MySQL side with `ssl_enforcement = "Enabled"` in the `azure_mysql.tf` file.

# Backup and recovery

## Backup

Due to Azure not doing a full backup of the MySQL server itself and its backups, we have created a cronjob to backup the full MySQL server.
See: https://docs.microsoft.com/en-us/azure/mysql/concepts-backup
```
Deleted servers cannot be restored. If you delete the server, all databases that belong to the server are also deleted and cannot be recovered. To protect server resources, post deployment, from accidental deletion or unexpected changes, administrators can leverage management locks.
```

The MySQL database is backed up every day by using a cronjob that starts the Argo workflow. The Argo workflow will do a "restore" of the MySQL server which is basically a copy of the server to a new resource group at the given time.

This is to ensure that __if__ the MySQL server is acidentally deleted, we have a full backup to put in place. 

The Argo workflow is using the az cli `az mysql server restore` to do the backup. https://docs.microsoft.com/en-us/cli/azure/mysql/server?view=azure-cli-latest#az-mysql-server-restore

The functionality of this can be found in the Cronjob `cronjob.backup.yaml` and in the Argo workflow `backup-workflow.yml`.

## Recovery

If the MySQL server is lost, a recovery process has been put in place.

Using the Make target `make list_backups REMOTE_ENV=develop` you will get a list of backus in the backup resource group. This specific resource group, is where we keep the backups. There is 1 for each `dev` and `prod`.

Once you use the `list_backups` target, you will get a list of possible MySQL servers to recover from. All are marked with dates.
```
===============> list_backups step initiated
vault-develop-v1-09-07-2019
vault-develop-v1-10-07-2019
vault-develop-v1-11-07-2019
vault-develop-v1-12-07-2019
```
In this case, we would like to recover from the latest possible.
Another make target then handles the restore from a backup and create a new MySQL server used by Vault.
We will need to specify the source server/backup where to recover from and use the target 
`make recover_mysql REMOTE_ENV=develop SOURCE_SERVER=vault-develop-v1-12-07-2019`

Notice we specified the `SOURCE_SERVER` to recover from with `vault-develop-v1-12-07-2019` we got earlier from the `list_backups` target.

Shortly after, the MySQL server will then be restored into the original resource group where the MySQL server was lost. Since the connection strings and DB are the same, Vault will after some time (5-10 minutes) recover pods in `crashloopbackoff` since they will have connection to a MySQL server again. It is also possible to restart the Vault pods to see an effect quicker.

# Database runbook

## Backup

Azure MySQL databases are backed up automatically on a periodic basis. Per [Azure documentation](https://docs.microsoft.com/en-us/azure/mysql/concepts-backup), the transaction log is backed up every five minutes, differential backup is performed twice a day and a full database backup is performed every week. The default retention period for backups is 7 days.

However, the default backup retention period can be changed with the following command.
```
az mysql server update --name mydemoserver --resource-group myresourcegroup --backup-retention 10
```

### Manual Backup (data dump)

Manual backup for Azure MySQL databases follows the same procedure as in the case of regular MySQL database servers. The recommended procedure is full export and reload. For detailed steps, see Azure documentation [here](https://docs.microsoft.com/en-us/azure/mysql/concepts-migrate-dump-restore)

Backup and restore of individual tables is not recommended for Vault as the tables are managed by Vault and the data is encrypted.

## Restore

Azure MySQL database instances can be restored to a `restore point` closest to the time specified in the restore command. The backup data is copied to the  restored server leaving the original database server untouched. See [documentation](https://docs.microsoft.com/en-us/azure/mysql/howto-restore-server-cli#server-point-in-time-restore) for more details.
```
az mysql server restore --resource-group myresourcegroup --name mydemoserver-restored --restore-point-in-time 2018-03-13T13:59:00Z --source-server mydemoserver
```

## Restore manual backup (data dump)
The backup file generated in the above backup step can be restored to a new database instance following the instructions [here](https://docs.microsoft.com/en-us/azure/mysql/concepts-migrate-dump-restore#restore-your-mysql-database-using-command-line-or-mysql-workbench).

Additionally, Azure MySQL database also supports Azure Database Migration Service to perform online migration of database instances. For more information, see documentation for [Data Migration Service](https://docs.microsoft.com/en-us/azure/dms/tutorial-mysql-azure-mysql-online)
