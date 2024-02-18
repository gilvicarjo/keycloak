# keycloak

# Setup Keycloak
# Setup Database
# Setup Webserver
# Migração Database

## Realms migration
Rule: Create realms from scratch on the new Keycloak instance.

In the database the Realms as stored in the table realm, be aware of the new Realm IDs created which you will need for the next steps.

## Clients migration
Rule: When creating the new Realms, Keycloak automatically create some default resources, one of them are the clients. Be aware of the new client IDs for this default initial clients, which you will need for the next steps.

The idea is to migrate only the customized clients, and the good news here is, once migrated they will remain with the same client_id.

The users are linked with clients by the service_account_client_link column which is the same client_id column in client table.

**Yes, after the migration of users and clients, the users that have link with default clients will need to have this registers updated or recreated.**

## Users migration
Users are stored in the user_entity table. They only need to set the column realm_id for the new realm ID created in the previous step.

## Credentials migration
The credential table is really simple it's a source of truth and only the table user_entity (Users) has link with it.

The column id in the user_entity must match the column user_id in credential table.

## Groups migration

Groups are stored in the table keycloak_group

Each register only refers for a upstream folder (or not) and a realm_id.

So the idea here in Groups is the same for Users, It just need to have the realm_id updated for the new ones in the new Keycloak setup.




  
