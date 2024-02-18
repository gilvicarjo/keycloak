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

### Customized Clients
The idea is to migrate only the customized clients, and the good news here is, once migrated they will remain with the same client_id.

The users are linked with clients by the service_account_client_link column which is the same client_id column in client table.

### Default Clients
**Yes, after the migration of users and clients, the users that have link with default clients will need to have this registers updated or recreated.**

## Users migration
Users are stored in the user_entity table. They only need to set the column realm_id for the new realm ID created in the previous step.

**One thing is missing here: verify the accounts.**

## Credentials migration
The credential table is really simple it's a source of truth and only the table user_entity (Users) has link with it.

The column id in the user_entity must match the column user_id in credential table.

## Groups migration

Groups are stored in the table keycloak_group

Each register only refers for a upstream folder (or not) and a realm_id.

So the idea here in Groups is the same for Users, It just need to have the realm_id updated for the new ones in the new Keycloak setup.

After migrate Users and Groups keeping theirs source Ids, now It's important to migrate the table user_group_membership. The name explains itself.

## Roles migration

The roles are stored in keycloak_role table. 

### Customized Roles (167 cases)
This query give us the customized roles 

```
SELECT id, client_realm_constraint, client_role, description, "name", realm_id, client, realm
FROM public.keycloak_role
and remove the ones that has variables refereces in description column.
```
So, this table has columns
client_realm_constraint: that can refer to the Realm or the client with this role.
When it refers to clients, no worries, you already migrate the clients in the step before! It will be there waiting for this role to be attached.
When it refers to the Realm, you only need to replace it with the new Realm ID.

#### Inside the Realm, How is a User associated with a Role?

A: by the table user_role_mapping. So let´s migrate this table too.

And this query will bring this for us:

```
SELECT role_id, user_id
FROM public.user_role_mapping
where role_id in (''id','id2','id3',...)
```
os ids, sao todos os ids da tabela keycloak_role

With that, we will now apply a query like that:

```
INSERT INTO public.user_role_mapping (role_id,user_id) VALUES
	 ('role_id1','user_id1'),
	 ('role_id1','user_id2'),
```


### Default Roles (397 cases)





  
