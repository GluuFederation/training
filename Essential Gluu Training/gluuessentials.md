# Essential Gluu Training
## Welcome!
Thank you for participating in our Essential Gluu Training course. The purpose of this course is to demonstrate the most commonly used features of the Gluu Server with concrete examples and hands-on participation.

Over the next two days, you will install and configure a fully functional Gluu Server with a populated database of users, both outgoing and incoming single sign-on capability, secure two-factor authentication, and basic API management. 

When we finish, you’ll be ready to leverage our server in your own production environment for strong authentication and policy management.

## About Gluu

## Essentials Installation
### Initial Configuration
Once you have SSHed into the provided virtual machine, you are ready to begin setting up your fully functional Gluu Server environment.

First, go ahead and note your IP. You’ll need this for several steps during this training, so make sure you have it onhand. The easiest way to do so is to use the `ifconfig` command. You’ll see your IP under “inet addr”. See an example `ifconfig` output here:

For many functions, including installation, the Gluu Server requires a Fully Qualified Domain Name (FQDN). Localhost is not supported. You can modify your computer’s hosts file to read your VM’s IP as a FQDN.  Navigate to `/etc` and open `hosts` with your favorite text editor. Add `<yourIP> example.gluu.org` under the localhost entry, like the following. Save and close the hosts file.

We also need an application called Dnsmasq to get full utility out of this domain name. You can get it pretty simply with git by entering the following commands:

```
sudo apt-get install git
git clone git://thekelleys.org.uk/dnsmasq.git
```

That’s it!

### Gluu Server Installation

Enter these commands to install the Gluu Server

|   |Description | Command |
|---|------------------------|---------|
|  1|Add Gluu Repository|`echo "deb https://repo.gluu.org/ubuntu/ trusty main" > /etc/apt/sources.list.d/gluu-repo.list`|
|  2|Add Gluu GPG Key| `curl https://repo.gluu.org/ubuntu/gluu-apt.key \| apt-key add -` |
|  3|Update/Clean Repo|`apt-get update`|
|  4|Install Gluu Server|`apt-get install gluu-server-3.1.3`|
|  5|Start Gluu Server chroot|`service gluu-server-3.1.3 start`|
|  6|Log into chroot|`service gluu-server-3.1.3 login`|
|  7|Navigate to setup|`cd /install/community-edition-setup`|
|  8|Run setup|`./setup.py`|

After running setup.py, you’ll get some options for what to install with the Gluu Server. For this training, choose the following options:

Do you acknowledge that use of the Gluu Server is under the MIT license? Yes
Enter IP Address [Default]
Enter hostname [example.gluu.org]
Enter your city or locality [anything]
Enter your state or province two letter code [anything]
Enter two letter country code: [anything]
Enter Organization name: [anything]
Enter email address for support at your organization [anything]
Optional: enter password for oxTrust and LDAP superuser [default or change]
Install oxAuth OAuth2 Authorization Server? [Yes]
Install oxTrust Admin UI? [Yes]
Install LDAP Server? [Yes]
Install [(1) Gluu OpenDJ]
Install Apache HTTPD Server [Yes]
Install Shibboleth SAML IDP? [No]
Install Asimba SAML Proxy? [No]
Install oxAuth RP? [Yes]
Install Passport? [Yes]
Install JCE 1.8? [Yes]
You must accept the Oracle Binary Code License Agreement for the Java SE Platform Products to download this software. Accept License Agreement? [Yes]

We will go further in depth with each of these options later on in this training.

Check out your oxTrust GUI by navigating to https://example.gluu.org in your browser. You’ll be notified that your browser doesn’t trust the certificates. This is fine, it’s because they’re self-signed. Log in with credentials `admin` and `secret`, respectively. If you see the following screen, you’re in business:

With that working, let’s go ahead and turn the Gluu Server off while we finish installing the rest of our required software. Enter the following commands (assuming you’re still logged into the chroot):
```
logout
service gluu-server-3.1.3 stop
```
### Lua-resty-openidc Nginx Library
Later, we will configure the Gluu Server for OpenID Connect. For this, we need to install a piece of software called lua-resty-openidc. This is a simple example app that uses the OpenID Connect authorization flow to transfer the user to a sample success message. We’ll go ahead and install everything we need to make it run now, but will configure it later.

Normally, this library is hosted on a different server than the main Gluu Server, but for testing purposes, we will modify it slightly to run on the same Virtual Machine. These modifications will be **bolded** and do not need to be performed in a production environment. They simply prevent the library from conflicting with the Gluu Server.

First, you need to install OpenResty 1.11.2.5, which will install most of the dependencies we’ll need. To do so, enter the commands in your terminal:

|1| `apt update`|
|2| `apt-get install gcc libssl-dev libpcre3 libpcre3-dev make`|
|3| `wget https://openresty.org/download/openresty-1.11.2.5.tar.gz`|
|4| `tar -xvf openresty-1.11.2.5.tar.gz`|
|5| `cd openresty-1.11.2.5`|
|6| `./configure -j2`|
|7| `make -j2`|
|8| `sudo make install`|

Then, add the OpenResty bin to PATH with this command:
```
export PATH=/usr/local/openresty/bin:$PATH
```
Now, download the remaining lua-resty dependencies with OPM (OpenResty Package Manager)

|1| `opm install bungle/lua-resty-session`|
|2| `opm install pintsized/lua-resty-http`|
|3| `opm install zmartzone/lua-resty-openidc`|

Next, we need to create some SSL certificates. On a normal configuration, these will be created on the Relying Party server, rather than the Gluu Server, but for this example, they’re the same server. Enter these two commands:

|1|`mkdir -p /usr/local/openresty/nginx/ssl/`
|2|`sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /usr/local/openresty/nginx/ssl/nginx.key -out /usr/local/openresty/nginx/ssl/nginx.crt`|

You’ll be prompted to provide some information for the certificate content. How you fill them out doesn’t matter, for this example. These certificates aren’t actually being sent outside of the VM.

Before we move on, we need to disable OpenResty to allow for configuration edits. The simplest way to kill it is to enter the following commands:
```
fuser -k 80/tcp
```
That’ll kill any process listening on port 80, which in this case is just OpenResty.

The last bit of configuration we need for the application is to alter OpenResty’s Nginx. Navigate in the terminal to `/usr/local/openresty/nginx/conf/` and open `nginx.conf` with your favorite text editor. 

You’ll need to change the .conf to the following:
```
events {
  worker_connections 1024;
}

http {

  lua_package_path "/usr/local/openresty/?.lua;;";

  resolver **127.0.0.1**;

  lua_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
  lua_ssl_verify_depth 5;

  # cache for discovery metadata documents
  lua_shared_dict discovery 1m;
  # cache for JWKs
  lua_shared_dict jwks 1m;

  server {
    listen **81** default_server;
    server_name _;
    return 301 https://$host$request_uri;
  }
  server {
    listen **444** ssl;

    ssl_certificate /usr/local/openresty/nginx/ssl/nginx.crt;
    ssl_certificate_key /usr/local/openresty/nginx/ssl/nginx.key;

    location / {

      access_by_lua_block {

          local opts = {
             redirect_uri_path = "/welcome",
             discovery = "https://***example.gluu.org***/.well-known/openid-configuration",
             client_id = "***$INUM***",
             client_secret = "***$SECRET***",
             ssl_verify = "no",
             scope = "openid email profile",
             redirect_uri_scheme = "https",
          }

          -- call authenticate for OpenID Connect user authentication
          local res, err = require("resty.openidc").authenticate(opts)

          if err then
            ngx.status = 500
            ngx.say(err)
            ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
          end

          ngx.req.set_header("X-USER", res.id_token.sub)
      }
    }
  }
}
```
Again, the **BOLDED* highlights only need to be changed for this training to allow both the Gluu Server and the RP to work on the same server. See the full doc to find the unmodified text block and full instructions.

The ***BOLDED AND ITALICIZED*** entries do need to be replaced with the inum and client secret that you will generate when you configure the RP entry in the Gluu Server later. We went ahead and replaced ${Gluu Server} with example.gluu.org, since we already know the hostname.

Our last step in this process is to start the Gluu Server back up. Again, just use the following command:
```
service gluu-server-3.1.3 start
```
For now, you don’t need to log back into the chroot. Once it’s running, you should be able to access the oxTrust GUI again at https://example.gluu.org.

Now that we have everything installed, we are ready to set everything up to make a custom, fully functional Gluu Server. Let’s start by talking about exactly what the Gluu Server is, and what it does.

## What is the Gluu Server?

The Gluu Server is a Chroot-based container distribution of free open source (FOSS) software for identity and access management (IAM). The server can be leveraged for user authentication, identity information, and policy management. The software inside is integrated together to provide a wide variety of IAM solutions, such as single sign-on (SSO), mobile authentication, API access management, two-factor authentication (2FA), and more.

Possibly the most important part of the Gluu Server is that it is free open source software. The code is available on GitHub,  Docker and Kubernetes documentation and deployment strategies are available, binary packages are published for Linux, and Gluu provides both free and VIP support options for the community. 

### What’s inside?

When you installed the Gluu Server today using the guide provided, you elected to install most of the software bundled within the Gluu Server distribution. The following is a brief explanation of what each of those software components do:

- **oxAuth**: The core of the Gluu Server, oxAuth serves as the OAuth 2.0 Authorization Server, the OpenID Connect Provider, and the UMA Authorization Server.
- **oxTrust**: This is the user interface for the Server. With few exceptions, everything you need to configure and operate your implementation is easily accessible here. 
- **Shibboleth IDP**: To support outbound SAML single sign-on (SSO), Shibboleth is included as a SAML identity provider.
- **Passport**: Passport is a versatile web application that the Gluu Server uses to support social login, inbound SAML, and inbound OpenID Connect SSO.
- **OpenDJ**: This LDAP directory server is where all Gluu user data, session data, and more are stored.
- **Apache2 Web Server**:The Apache Web Server brings HTTP services to the Gluu Server.
- **JCE 1.8**: The Java Cryptography Extension provides encryption and key generation capabilities to the Gluu Server.
- **oxAuth RP**: This is an app that provides sample requests and responses for OpenID Connect operations.

## Synchronizing users from Active Directory/external LDAP server

LDAP, or Lightweight Directory Access Protocol, is a database protocol that uses a tree format to facilitate fast and efficient lookups. With the hierarchical tree format, searches follow a path through the database that ignores any data that is not relevant to your search. One of the key benefits of LDAP over other database solutions is that LDAP is relatively vendor-agnostic. LDAP has a text-based format called "LDIF", or LDAP Data Interchange Format, that allows for data to always be exported from one LDAP server to another. 

###	LDAP structure
An individual unit of information in LDAP is called an “entry.” An LDAP directory is composed of many entries, connected together to form a tree. These entries are located using an address system using Distinguished Names (DNs), which are made up of Relative Distinguished Names (RDNs). The DN is the full address of an entry in the tree, while the RDN is a partial path relative to another entry. For example, the following DN:
```
uid=foo,ou=people,o=acme
```
is composed of three RDNs, each separated by a comma (but not a space).

Every DN must be unique in the tree, as it is the full address of where to find a particular entry. Attempting to add a duplicate DN will result in an error.

The tree itself is known as the namespace or Directory Information Tree (DIT), and is defined by how we name each entry in the tree. The following chart is the namespace for a hypothetical company, Acme, and consists of three levels.
.


The prefix of each RDN on this chart defines its location in the hierarchy. Spelled out, they are domain component (dc), organizational unit (ou), common name (cn), and user identification (uid). The first level, at the top of this chart, is called the root node. In this case, it consists of one entry that is made up of two components, dc=acme,dc=com. The second level consists of two entries, ou=people and ou=groups, which are both subordinate to `dc=acme,dc=com`. The third section, the leaf entries, contains four entries: two people and two groups.

Each entry is composed of a DN and data, in that order. If you look at the following LDIF for the `uid=foo` entry from the previous chart, you’ll see how it’s structured. 



The first line is the DN, as previously mentioned. Following that are four object classes, which are predefined in a schema to require or optionally include certain attributes, which follow the object classes. For this example, the object class “person” requires the “cn” and “sn” attributes. In general, it’s best to use as few required attributes as possible. Optional attributes allow for greater flexibility. 

### Gluu’s Internal LDAP   

As you saw when installing the Gluu Server, we give the option of installing one of two LDAP servers: OpenDJ or OpenLDAP. Both these LDAP servers have been modified by Gluu to integrate with the software bundled in the server. If you are using a single, unclustered Gluu Server, either will serve your purposes. For this training, we are installing OpenDJ, which tends to be more popular, especially as it’s improved replication works better with the clustered Gluu Servers that are becoming more prominent among our customers.

### LDAP Synchronization

Using Cache Refresh, Gluu Server administrators can synchronize Gluu’s internal LDAP database with one or more external LDAP servers or Microsoft’s Active Directory. Syncing from a standalone backend server speeds up authentication transactions, and it can allow attribute transformations such as changing the attribute name or using an interception script to change values. These transformations are stored in the Gluu LDAP service.

### Configuring Cache Refresh

For this training, we have provided a sample external LDAP server that is populated with a collection of example user information to allow for a live demonstration of the functionality.

First, bring up your oxTrust UI, then navigate to `Configuration > Cache refresh` on the sidebar. 

We need to configure three of the four tabs at the top of the page, starting with `Customer Backend Key/Attributes`. This tells Cache Refresh which information to pull from the external LDAP server. Set the following values:


This tells Cache Refresh to pull entries with the uid RDN, person object class, and mail, cn, and sn attributes from the external LDAP into the internal LDAP. Hit the “Update” button at the bottom, and we’ll move on.

Next, we’ll move to the Source Backend LDAP Servers tab. This tells Cache Refresh where to find the external LDAP server. You’ll configure it as follows:

We used the name “source”, although this can be anything, it’s just to keep track of which external server you’re pulling from, as more than one server can be designated for failover.

The fourth tab, Inum LDAP Server, we don’t actually need to touch. It’s set by default to the internal LDAP that you are synchronizing with the sample external LDAP.

Finally, we’ll navigate back to the first tab, Cache Refresh. This tab dictates what happens during synchronization. The exact details of this page will change, as it uses details from your particular VM. Ignoring the “last run” section at the top, set up the page as follows: 

Refresh Method: leave this as “Copy”
Add source attribute to destination attribute mapping: This translates attribute names from the external LDAP to the internal. We’re using the same attributes, so create three mappings, as follows:
Polling interval (minutes): This is how often Cache Refresh pulls information from the external LDAP. For production, we recommend at least 15 minutes, but for this training, let’s do 5 so we can quickly see that it worked
Server IP Address: This is the IP for your particular VM. For this training, we’ll provide this for you.
Snapshot Folder: Leave default
Snapshots count: Leave default
Keep external persons: Leave default
Load source data with limited search: Leave default
Search size limit: Leave default
Cache Refresh: Select “Enable”

Then, just hit Update, and it’s ready to go!
            Testing the LDAP synchronization
A handy method to test if Cache Refresh worked properly is included in oxTrust. After the five minute polling interval we set in the previous section, the “Last Run” section that we previously ignored will be populated with information on when the last synchronization occurred, as well as how many entries were updated. 

If the “Problems at the last run” entry is more than zero, that means that some entries were rejected. If this occurs when setting up Cache Refresh on your own server after this training, contact Gluu Support so we can help figure out what went wrong.
To make sure that the updates came through properly, check a sample user by navigating to “Users > Manage People” and searching for “test”. Click on the result’s UID field, and you’ll see all the data for the sample user.

Congratulations! Your Gluu Server is now populated with users!

## Two-factor authentication (2FA)

The weakest part of any authentication solution is the password. Over half of internet users use the same password for all applications, so a single leak could compromise every service that user has an account with. Even with a strong, unique password, the user is susceptible to phishing, scams, man-in-the-middle attacks, and more.

While there is no way to fully protect the user from compromised authentication credentials, they can certainly be strengthened through the addition of multiple factors of authentication. Out of the box, the Gluu Server includes interception scripts to easily configure several different versions of 2FA. We’ll talk more about interception scripts in a moment.

As an introduction to multifactor authentication, there are a few different types of authentication factors.

Something you know: This is the traditional authentication factor, including not only the password and PINs but also the security questions that are often used to recover a lost password. While these should be kept secret, someone snooping on social media or using phishing techniques can easily bypass these factors.
Something you have: This is a physical item that helps verify that you are who you say you are. This category includes smart cards, hand-held tokens, one-time passwords, etc. Typically, other than it being the same piece of hardware that is registered to your account, there is nothing inherently about the hardware that proves who is holding it. That makes it susceptible to theft and man-in-the-middle attacks, if not combined with one or more other forms of authentication.
Something you are: This category refers to biometric authentication, including fingerprints, hand geometry, retinal scans, and even handwriting or voice analysis. These factors are difficult to faithfully reproduce, but still vulnerable to clever workarounds. Additionally, biometric scans can, depending on the sensitivity of the system, be prone to false rejection and false approval errors.

Each of these factors, individually, has easily exploited weaknesses. Combining two or more helps mitigate these vulnerabilities to provide the most secure authentication method possible. Although multi-factor systems have existed for decades and are becoming more and more prominent for common users, probably the most recognizable example is the debit card. Generally, a debit card is only usable when combining something you have, the physical card, and something you know, the PIN. 

### How does the Gluu Server handle 2FA?

The Gluu Server uses a parameter called “Authentication Context Class Reference”, or “acr” to specify what type of authentication happened. This parameter is supported by both OpenID Connect and SAML for the client to signal to the authentication provider the preferred method to authenticate the subject. 

Organizations that use the Gluu Server have a wide array of authentication requirements--not only how to authenticate the person, but also how to log, how to detect fraud, to gather consent, and to update personal data records. With such varied requirements by our customers, there would be no way to build a single GUI to give all the flexibility their organizations require. Instead, Gluu created an authentication “interception script”--a standard interface that allows developers to code the exact workflow they need. This allows the Gluu Server to be extended without forking the Gluu Server code, and therefore making the server more difficult to keep up with updates and upgrades. Gluu uses interception scripts for several tasks, but the one we’ll focus on here is called “Person Authentication.”

When you install the Gluu Server, password authentication is the default, as seen in the oxTrust interface here:

### What forms are supported?

By default, the Gluu Server comes packaged with interception scripts for several forms of authentication.
- Basic: Basic authentication is simply checking usernames and passwords against the local Gluu LDAP. Since we configured an external LDAP previously with Cache Refresh, the internal LDAP is updated with user entries.
- U2F: FIDO Universal 2nd Factor (U2F) is an open authentication standard that strengthens and simplifies 2FA using specialized USB or NFC devices.
- Super Gluu: Super Gluu is a free 2FA mobile app built to work with the Gluu Server.
- OTP apps: The OTP interception script allows for the use of any OTP app as the second authentication factor.
- SMS OTP: The Twilio SMS OTP script, in combination with a Twilio account, allows the Gluu Server to use an OTP sent via text message to authenticate the user.
- Duo Security: The Duo interception script allows for the Duo SaaS authentication provider to be used as the second factor, after username and password login.
- Certificate Authentication: Using the Cert authentication method, the Gluu Server can use browser certificates or a smart card as the first form of authentication, and username and password for the second.

### Configuring a supported 2FA script

Actually setting up the Gluu Server in oxTrust to use a specific form of 2FA is extremely simple, especially if that format is packaged with the Gluu Server by default. All you need to do is navigate to Configuration > Manage Custom Scripts. You’ll see the following:



Drop down the arrow to the right of your 2FA selection. Scroll down to the bottom of that section, and click the checkbox.



Scroll all the way to the bottom of the page and click the Update button.

Your server will now accept the selected form of 2FA as an authentication method.

To set it as your default form, navigate to `Configuration > Manage Authentication > Default Authentication Method`.



Change both “Default acr” and “oxTrust acr” to the chosen authentication form, and click update. As simple as that, your Gluu Server is configured for 2FA.

###	Custom Scripts

## SSO using SAML and OpenID Connect

### SAML

SAML, the Security Assertion Markup Language, was one of the first important solutions to allow a user to authenticate once and access websites both inside and outside their organization. SAML is an XML standard that was developed by a large group of interested parties--29 organizations and several individuals at OASIS, a nonprofit consortium that provides support for the development, convergence and adoption of open standards. It works by sending a signed, verifiable statement that demonstrates that the person requesting access is who they claim to be.

SAML is actually not, itself, an authentication protocol, it’s a federation protocol. The actual authentication method is outside of its scope, it instead redirects the person to an identity provider.

SAML is a mature standard that is relatively stable--it hasn’t been significantly updated since its 2.0 inception. However, as it was finalized before the age of developer-friendly APIs, ease of use was not a design goal. 

### SAML Assertions

The terms defined by SAML have become an important part of the IAM vocabulary. For example, a “SAML Assertion” is a statement that is written in XML and issued by an “Identity Provider” about a “Subject” (person) for a “Relying Party” (the recipient of the assertion), who is normally a “Service Provider” (website). Assertions contain contextual information about the authentication procedure, as well as “Attributes”--similar to LDAP attributes, these are little pieces of information about the person, such as first name or last name.

A SAML assertion can be composed of four sections:
- Subject is an identifier for the person. This can be a one-time identifier that will change each time the person visits the site, or it can be a consistent identifier that will enable continuity with a person’s previous activity.
- Authentication Statements contain information about who this person is (i.e. his username in the IDP) and when and how the person was authenticated.
- Attribute Statements contain information about the subject, like first name, last name, email address, role or group memberships.
- Authorization Statements contain information about whether the subject should be granted access to a requested resource. This is a somewhat esoteric part of SAML, which you will probably not encounter in the wild.

The following is an example SAML assertion sent from an IDP (in this case, idp.example.org) to an SP (in this case, sp.example.com) that contains both an Authentication and Attribute: Statement, abbreviated to increase readability.



The Gluu Server does come packaged with Shibboleth, which is a fully functional SAML IDP. However, for the purposes of this training, we are going to use a different protocol, OpenID Connect. If you’d like to see more about Shibboleth’s role in the Gluu Server, see our docs.

### OpenID Connect

OpenID Connect (or just Connect) is an extension of the OAuth2 standard that brings enhanced usability and functionality to compete with SAML. Connect was developed after Google, Microsoft, Facebook, and other large consumer identity provider services had already introduced their own OAuth2 APIs, which were processing millions of transactions per day. All this data on security, developer useability, and end-user behavior was taken into account to greatly benefit the final product. A switch from XML data structures to JSON, combined with RESTful web services are both more efficient on the wire and require less computing power for small devices, increasing functionality with mobile use cases. Rather than focusing on enterprise use cases like SAML, OpenID Connect was designed to address a range of privacy and security requirements, from non-sensitive information to highly secure transactions. Overall, the main design goal was to keep simple things simple, but make complex things possible.

OpenID Connect has many parallels to SAML. The OpenID Provider (“OP”) is the equivalent of the SAML IDP--the software component that authenticates the subject, and returns an assertion to the client or Relying Party (or “RP). The RP is a rough functional equivalent of the SAML SP. While in SAML we might call pieces of information about a subject “attributes”, in Connect, we call these things “user claims.” The equivalent of the SAML assertion (potentially signed and encrypted XML), is an id_token, a signed and potentially encrypted JSON Web Token, or JWT (pronounced "jot").

###Testing SSO with Nginx lua-resty-openidc

To test our OpenID Connect authentication flow, we need to finish configuring our lua-resty-openidc RP that we installed at the beginning of the training. To do so, go into oxTrust and click `OpenID Connect > Clients`. At the top of the screen, click Add Client. You’ll see the following screen:



The Client Name can be anything you want it to be, preferably human-readable for when multiple RPs are authenticating against the same Gluu Server. Client Description can be more thorough, describing the purpose of the client. 

Client Secret is essentially that password that’s present along with the JWT. Normally, you want this secret to be highly secure, perhaps by generating it with this command in a terminal:
```
gpg --gen-random --armor 1 30
```
For the purposes of this demonstration, however, it can be something like “password” or “secret”, since we aren’t actually protecting anything. Take note of whatever secret you use, as we’ll add it to the nginx.conf file in a bit.

We’re going to skip most of the other configuration for simplicity’s sake and move all the way to the bottom of the screen. We need to input Add Login Redirect URI, Add Scope, Add Response Type, and Add Grant Type.

For our example, our Redirect Login URI will be “https://example.gluu.org:444/welcome”. In a live setting, when we have the RP properly on a separate server from the Gluu Server, the address will be the FQDM of the RP, such as “https://test.hostname.com/welcome”.

Next, click `Add Scope > Search`. Check “email”, “openid”, and “profile”.
Click `Add Response Type` and check “code” and “id_token”.
Finally, click `Add Grant Type` and check “authorization_code”.

Once this is done, simply click the “Add” button at the bottom of the page. Once the page refreshes, note the inum from the OpenID Connect/Clients dashboard next to the Display Name of the newly created client. We’ll need it in a moment.



For our last step, we will go back into the configuration file in your VM at `/usr/local/openresty/nginx/conf/nginx.conf` and replace the `client_id` and `client_secret` fields here:



These will be the inum and secret that you just noted while registering your client in oxTrust.

Once that’s finished, save the configuration file and run the `openresty` command in your terminal to start up the RP. 

To test that the RP is authenticating against the Gluu Server, navigate to “https://example.gluu.org:444”. Again, if you were using a separate server as the RP, it would simply be the FQDN from that server, in our previous example “https://test.hostname.com”.

You’ll be redirected to a success page, and your OpenID Connect RP is working properly!

## Supporting social login and inbound identity

### Passport and Inbound Identity
Coming soon

## API access management

## Best Practices

### Back Up the Gluu Server
To ensure continuity and mitigate failure, we recommend backing up the Gluu Server frequently. Our standard suggestion is at least one daily and one weekly backup.

There are several ways to back the server up. One simple and rapid method is to simply use the snapshot feature of whatever virtual machine you are using. For instance, Digital Ocean has Live Snapshot and Droplet Snapshot, while VMWare has Snapshot manager, and so on. These snapshots should be taken for all Gluu environments (such as production, development, QA, and so on) and tested periodically to confirm consistency and integrity.

Another method is to tarball the entire Gluu Server chroot folder using the tar command in the terminal. To do so, follow these steps:

|   |Description|Command|
|---|---|---|
|1| Stop the server |`service gluu-server-3.1.3 stop` |
|2| Make a backup | `tar cvf gluu313-backup.tar /opt/gluu-server-3.1.3/`|
|3|Restart the server | `service gluu-server-3.1.3 start` |

###	Back Up LDIF Data
In addition to the general server backup, you should also separately back up your LDIF periodically. Since the data is saved in plain text, there are more recovery options than a binary backup. 

Since we’re using OpenDJ, the easiest way to do so is, from inside the chroot, to just stop the LDAP server and issue the following command, specifying your preferred directory and file:
```
/opt/opendj/bin/export-ldif -n userRoot -l /<path>/<to>/<backup>/<filename>.ldif
```
### Make oxTrust Local Only
Notes: You should have to VPN or SSH to get to admin console. In default installation, it’s internet facing for ease of use. /identity is the path.

## What’s Next for Gluu?


