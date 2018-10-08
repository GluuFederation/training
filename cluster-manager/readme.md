# Cluster Manager Training

Discuss high availability considerations with Gluu Server:

- A diagram of how communication happens between clients -> Gluu Server as well as Gluu Server -> Gluu Server.

    - Further detail showing how client requests to oxAuth are stateless, as all information necessary to process the request is present in the request itself. This is however now true with SCIM as additional stateful information is required to work effectively.

    - oxTrust cares about state and therefore needs each request from a specific identifier, usually an IP address, to be to the same oxTrust entity.

    - LDAP is queried by both oxTrust and oxAuth, and is therefore vital to the health of a cluster of Gluu Servers. 

    - In previous versions a separate shared cache of data needed to exist but that has been replaced by caching in LDAP in 3.1.4

- LDAP database replication and how it's enabled. (Refer to cheat sheet.)

- Shared cache (LDAP in 3.1.4 is least performant, but less complex.)

- Go over Configuration settings.

- Load balancing

- OpenDJ LDAP command cheat sheet.

    - Discuss the structure of inum's in Gluu LDAP

```
# Search for all 'cn=' entries:

/opt/opendj/bin/ldapsearch -p 1636 -Z -X -D 'cn=directory manager' -w secret -b o=gluu "cn=*"

# Simple way to check entry count inside of LDAP:

/opt/opendj/bin/ldapsearch -p 1636 -Z -X -D 'cn=directory manager' -w secret -b o=gluu "objectclass=*" | wc -l

# Search all schema:

/opt/opendj/bin/ldapsearch -p 1636 -Z -X -D 'cn=directory manager' -w secret -b cn=schema -s base "objectclass=*" +

# Check replication status

/opt/opendj/bin/dsreplication status -n -X -h -p 1444 -I admin -w secret

# List all servers in replication topology

/opt/opendj/bin/dsconfig list-replication-domains --provider-name "Multimaster Synchronization" --port 4444 --bindDN "cn=Directory Manager" --bindPassword secret --no-prompt --trustAll

# Enforce uniqueness on the "uid" attribute:

/opt/opendj/bin/dsconfig set-plugin-prop --hostname localhost --port 4444 --bindDN "cn=Directory Manager" --bindPassword secret --plugin-name "UID Unique Attribute" --set base-dn:'ou=people,o=@!82F6.00B8.C20E.272F!0001!DC79.0594,o=gluu'  --set enabled:true --trustAll --no-prompt
```

- Operations Guidance:

    - For example https://github.com/GluuFederation/cluster-mgr/wiki/Backup-and-Restore-LDAP-Data-Online
    - Adding valid certificates manually (automated feature pending).

================================================================ 
