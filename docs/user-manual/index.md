---
layout: page
title: User Manual
advertise: User Manual
---

Version 2.1rc1, released 2017-05-05

This document has all the information you need to get up and
running with User Sync. It presumes familiarity with the use of
command line tools on your local operating system, as well as a
general understanding of the operation of enterprise directory
systems.

### Table of Contents
{:."no_toc"}

* TOC Placeholder
{:toc}

## Introduction

User Sync, from Adobe, is a command-line tool that moves user and
group information from your organization's LDAP-compatible
enterprise directory system (such as Active Directory) to the
Adobe User Management system.

Each time you run User Sync it looks for differences between the
user information in the two systems, and updates the Adobe
directory to match your directory.

### Prerequisites

You run User Sync on the command line or from a script, from a
server that your enterprise operates, which must have Python
2.7.9 or higher installed. The server must have an internet
connection, and be able to access Adobe's User Management system
and your own enterprise directory system.

The User Sync tool is a client of the User Management API
(UMAPI). In order to use it, you must first register it as an API
client in the [Adobe I/O console](https://www.adobe.io/console/),
then install and configure the tool, as described below.

The operation of the tool is controlled by local configuration
files and command invocation parameters that provide support for
a variety of configurations. You can control, for example, which
users are to be synced, how directory groups are to be mapped to
Adobe groups and product configurations, and a variety of other
options.

The tool assumes that your enterprise has purchased Adobe product
licenses. You must use the
[Adobe Admin Console](https://adminconsole.adobe.com/enterprise/) to define
user groups and product configurations. Membership in
these groups controls which users in your organization can access
which products.

### Operation overview

User Sync communicates with your enterprise directory through
LDAP protocols. It communicates with Adobe's Admin Console
through the Adobe User Management API (UMAPI) in order to update
the user account data for your organization. The following figure
illustrates the data flow between systems.

![Figure 1: User Sync Data Flow](media/adobe-to-enterprise-connections.png)

Each time you run the tool:

- User Sync requests employee records from an enterprise
directory system through LDAP.
- User Sync requests current users and associated product
configurations from the Adobe Admin Console through the User
Management API.
- User Sync determines which users need to be created, removed,
or updated, and what user group and product configuration
memberships they should have, based on rules you have defined in
the User Sync configuration files.
- User Sync makes the required changes to the Adobe Admin Console
through the User Management API.

### Usage models

The User Sync tool can fit into your business model in various
ways, to help you automate the process of tracking and
controlling which of your employees and associates have access to
your Adobe products.

Typically, an enterprise runs the tool as a scheduled task, in
order to periodically update both user information and group
memberships in the Adobe User Management system with the current
information in your enterprise LDAP directory.

The tool offers options for various other workflows as well. You
can choose to update only the user information, for example, and
handle group memberships for product access directly in the Adobe
Admin Console. You can choose to update all users, or only
specified subsets of your entire user population.
In addition, you can separate the tasks of adding and updating
information from the task of removing users or memberships. There
are a number of options for handling the removal task.

For more information about usage models and how to implement
them, see the [Usage Scenarios](#usage-scenarios) section below.

---

## Setup and Installation

The use of the User Sync tool depends on your enterprise having
set up product configurations in the Adobe Admin
Console. For more information about how to do this, see the
[Configure Services](https://helpx.adobe.com/enterprise/help/configure-services.html#configure_services_for_group)
help page.

### Set up a User Management API integration on Adobe I/O

The User Sync tool is a client of the User Management API. Before
you install the tool, you must register it as a client of the API
by adding an *integration* in the Adobe I/O
[Developer Portal](https://www.adobe.io/console/). You will need
to add an Enterprise Key integration in order to obtain the
credentials the tool needs to access the Adobe User Management
system.

The steps required for creating an integration are described in
detail in the
[Setting up Access](https://www.adobe.io/apis/cloudplatform/usermanagement/docs/setup.html)
section of the Adobe I/O User Management API website.  The
process requires that you create an integration-specific
certificate, which may self-signed.  When the process is
complete, you will be assigned an **API key**, a **Technical
account ID**, an **Organization ID**, and a **client secret**
that the tool will use, along with your cerficate information, to
communicate securely with the Admin Console. When you install the
User Sync tool, you must provide these as configuration
values that the tool requires to access your organization's user
information store in Adobe.

### Set up product-access synchronization

If you plan to use the User Sync tool to update user access to
Adobe products, you must create groups in your own enterprise
directory that correspond to the user groups and product
configurations that you have defined in the
[Adobe Admin Console](https://www.adobe.io/console/). Membership
in a product configuration grants access to a particular set of
Adobe products. You can grant or revoke access to users or to
defined user groups by adding or removing them from a product
configuration.

The User Sync tool can grant product access to users by adding
users to user groups and product configurations based on their
enterprise directory memberships, as long as the group names are
correctly mapped and you run the tool with the option to process
group memberships.

If you plan to use the tool in this way, you must map your
enterprise directory groups to their corresponding Adobe groups
in the main configuration file. To do this, you must ensure that
the groups exist on both sides, and that you know the exact
corresponding names.

#### Check your products and product configurations

Before you start configuring User Sync, you must know what Adobe
products your enterprise uses, and what product
configurations and user groups are defined in the Adobe User
Management system. For more information, see the help page for
[configuring enterprise services](https://helpx.adobe.com/enterprise/help/configure-services.html#configure_services_for_group).

If you do not yet have any product configurations, you can use the
Console to create them. You must have some, and they must have
corresponding groups in enterprise directory, in order to
configure User Sync to update your user entitlement information.

The names of product configurations generally identify
the types of product access that users will need, such as All
Access or Individual Product Access. To check the exact names, go
to the Products section in the
[Adobe Admin Console](https://www.adobe.io/console/) to see the
products that are enabled for your enterprise. Click a product to
see the details of product configurations that have been
defined for that product.

#### Create corresponding groups in your enterprise directory

Once you have defined user groups and product configurations in
the Adobe Admin Console, you must create and name corresponding
groups in your own enterprise directory. For example, a directory
group corresponding to a “All Apps” product configuration
might be called “all_apps”.

Make a note of the names you choose for these groups, and which
Adobe groups they correspond to. You will use this to set up a
mapping in the main User Sync configuration file. See details in
the [Configure group mapping](#configure-group-mapping) section
below.

It is a best practice to note in the description field of the Product Configuration or User Group that the group is managed by User Sync and should not be edited in the Admin Console.

![Figure 2: Group Mapping Overview](media/group-mapping.png)

### Installing the User Sync tool

#### System requirements

The User Sync tool is implemented using Python, and requires
Python version 2.7.9 or higher. For each environment in which you
intend to install, configure and run the script, you must make
sure that Python has been installed on the operating system
before moving to the next step. For more information, see the
[Python web site](https://www.python.org/).

The tool is built using a Python LDAP package, `pyldap`, which in
turn is built on the OpenLDAP client library. Windows Server,
Apple OSX and many flavors of Linux have an OpenLDAP client
installed out of the box.  However, some UNIX operating systems,
such as OpenBSD and FreeBSD do not have this included in the base
installation.

Check your environment to be sure that an OpenLDAP client is
installed before running the script. If it is not present in your
system, you must install it before you install the User Sync
tool.

#### Installation

The User Sync Tool is available from the
[User Sync repository on GitHub](https://github.com/adobe-apiplatform/user-sync.py). To
install the tool:

1. Create a folder on your server where you will install the 
User Sync tool and place the configuration files.

1. Click the **Releases** link to locate the latest release,
which contains the release notes, this documentation, sample
configuration files, and all the built versions (as well as
source archives).

2. Select and download the compressed package for your platform
(the `.tar.gz` file). Builds for Windows, OSX, and Ubuntu are
available. (If you are building from source, you can download the
Source Code package that corresponds to the release, or use the
latest source off the master branch.)

3. Locate the Python executable file (`user-sync` or
`user-sync.pex` for Windows) and place it in your User Sync
folder.

4. Download the `example-configurations.tar.gz` archive of sample configuration
files.  Within the archive, there is a folder for “config files –
basic”.  The first 3 files in this folder are required. Other
files in the package are optional and/or alternate versions for
specific purposes. You can copy these to your root folder, then
rename and edit them to make your own configuration files. (See
the following section,
[Configuring the User Sync Tool](#configuring-the-user-sync-tool).)

5. **In Windows only:**

    Before running the user-sync.pex executable in Windows, you might
need to work around a Windows-only Python execution issue:

    The Windows operating system enforces a file path length limit of
260 characters. When executing a Python PEX file, it creates a
temporary location to extract the contents of the package. If the
path to that location exceeds 260 characters, the script does not
execute properly.

    By default, the temporary cache is in your home folder, which may
cause pathnames to exceed the limit. To work around this issue,
create an environment variable in Windows called PEX\_ROOT, a set
the path to C:\\user-sync\\.pex. The OS uses this variable for
the cache location, which prevents the path from exceeding the
260 character limit.

6. To run the User Sync tool, run the Python executable file,
`user-sync` (or execute `python user-sync.pex` on Windows).

#### Security Considerations

Because the User Sync application accesses sensitive information
on both the enterprise and Adobe sides, its use involves a number
of different files that contain sensitive information. Great care
should be take to keep these files safe from unauthorized access.

User Sync release 2.1 or later allow you to store credentials in
the operating system's secure credential store as an alternative
to storing them in files and securing those files, or to store
umapi and ldap configuration files in a secure way that you can
define.  See section [Security recommendations](#security-recommendations)
for more details.

##### Configuration files

Configuration files must include sensitive information, such as
your Adobe User Management API key, the path to your certificate
private key, and the credentials for your enterprise directory
(if you have one). You must take necessary steps to protect all
configuration files and ensure that only authorized users are
able to access them. In particular: do not allow read access to
any file containing sensitive information except from the user
account that runs the sync process.

If you choose to use the operating system to store credentials,
you still create the same configuration files but rather than storing
the actual credentials, they store key ids that are used to look up
the actual credentials.  Details are shown in
[Security recommendations](#security-recommendations).

If you are having User Sync access your corporate directory, it
must be configured to read from the directory server using a
service account. This service account only needs read access and
it is recommended that it _not_ be given write access (so that
unauthorized disclousre of the credential does not allow write
access to whomever receives it).

##### Certificate files

The files that contains the public and private keys, but
especially the private key, contain sensitive information. You
must retain the private key securely. It cannot be recovered or
replaced. If you lose it or it is compromised, you must delete
the corresponding certificate from your account. If necessary,
you must create and upload a new certificate. You must protect
these files at least as well as you would protect an account name
and password. The best practice is to store the key files in a
credential management system or use file system protection so
that it can only be accessed by authorized users.

##### Log files

Logging is enabled by default, and outputs all transactions
against the User Management API to the console. You can configure
the tool to write to a log file as well. The files created during
execution are date stamped and written to the file system in a 
folder specified in the configuration file.

The User Management API treats a user’s email address as the
unique identifier. Every action, along with the email address
associated with the user, is written to the log. If you choose to
log data to files, those files contain this
information.

User Sync does not provide any log retention control or
management. It starts a new log file every day.  If you choose to 
log data to files, take necessary
precautions to manage the lifetime and access to these files.

If your company’s security policy does not allow any personally
identifiable information to be persisted on disk, configure the
tool to disable logging to file. The tool continues to output the
log transactions to the console, where the data is stored
temporarily in memory during execution.

### Support for the User Sync tool

Adobe Enterprise customers can use their normal support channels to
get support for User Sync.

Since this is an open source project, you can also open an issue in
GitHub.  To help with the debugging process, include your platform, 
command line options, and any log
files that are generated during the application execution in your
support request (as long as they contain no confidential
information).


---

## Configuring the User Sync Tool

The operation of the User Sync tool is controlled by a set of
configuration files with these file names, located (by default) in the same
folder as the command-line executable.

| Configuration File | Purpose |
|:------|:---------|
| user-sync-config.yml | Required. Contains configuration options that define the mapping of directory groups to Adobe product configurations and user groups, and that control the update behavior.  Also contains references to the other config files.|
| connector&#x2011;umapi.yml&nbsp;&nbsp; | Required. Contains credentials and access information for calling the Adobe User Management API. |
| connector-ldap.yml | Required. Contains credentials and access information for accessing the enterprise directory. |


If you need to set up access to Adobe groups in other organizations that
have granted you access, you can include additional configuration
files. For details, see the
[advanced configuration instructions](#accessing-groups-in-other-organizations)
below.

### Setting up configuration files

Examples of the three required files are provided in the `config
files - basic` folder in the release artifact
`example-configurations.tar.gz`:

```text
1 user-sync-config.yml
2 connector-umapi.yml
3 connector-ldap.yml
```

To create your own configuration, copy the example files to your
User Sync root folder and rename them (to get rid of the leading
number). Use a plain-text editor to customize the your copied
configuration files for your environment and usage model. The
examples contain comments showing all possible configuration
items. You can uncomment items that you need to use.

Configurations files are in [YAML format](http://yaml.org/spec/)
and use the `yml` suffix. When editing YAML, remember some
important rules:

- Sections and hierarchy in the file are based on
indentation. You must use SPACE characters for indentation. Do
not use TAB characters.
- The dash character (-) is used to form a list of values. For
example, the following defines a list named “adobe\_groups”
with two items in it.

```YAML
adobe_groups:
  - Photoshop Users
  - Lightroom Users
```

Note that this can look confusing if the list has only one item
in it.  For example:

```YAML
adobe_groups:
  - Photoshop Users
```

### Create and secure connection configuration files

The two connection configuration files store the credentials that
give User Sync access to the Adobe Admin Console and to your
enterprise LDAP directory. In order to isolate the sensitive
information needed to connect to the two systems, all actual
credential details are confined to these two files. **Be sure to
secure them properly**, as described in the
[Security Considerations](#security-considerations) section of
this document.

There are three techniques supported by User Sync for securing credentials.

1. Credentials can be placed in the connector-umapi.yml and connector-ldap.yml files directly and the files protected with operating system access control.
2. Credentials can be placed in the operating system secure credential store and referenced from the two configuration files.
3. The two files in their entirety can be stored securely or encrypted and a program that returns their contents is referenced from the main configuration file.


The example configuration files include entries that illustrate each of
these techniques.  You would keep only one set of configuration items
and comment out or remove the others.

#### Configure connection to the Adobe Admin Console (UMAPI)

When you have obtained access and set up an integration with User
Management in the Adobe I/O
[Developer Portal](https://www.adobe.io/console/), make note of
the following configuration items that you have created or that
have been assigned to your organization:

- Organization ID
- API Key
- Client Secret
- Technical Account ID
- Private Certificate

Open your copy of the connector-umapi.yml file in a plain-text
editor, and enter these values in the “enterprise” section:

```YAML
enterprise:
  org_id: "Organization ID goes here"
  api_key: "API key goes here"
  client_secret: "Client Secret goes here"
  tech_acct: "Tech Account ID goes here"
  priv_key_path: "Path to Private Certificate goes here"
```

**Note:** Make sure you put the private key file at the location
specified in `priv_key_path`, and that it is readable only to the
user account that runs the tool.

In User Sync 2.1 or later there is an alternative to storing the private key in a separate file; you can place
the private key directly in the configuration file.  Rather than using the
`priv_key_path` key, use `priv_key_data` as follows:

	  priv_key_data: |
	    -----BEGIN RSA PRIVATE KEY-----
	    MIIJKAIBAAKCAge85H76SDKJ8273HHSDKnnfhd88837aWwE2O2LGGz7jLyZWSscH
	    ...
	    Fz2i8y6qhmfhj48dhf84hf3fnGrFP2mX2Bil48BoIVc9tXlXFPstJe1bz8xpo=
	    -----END RSA PRIVATE KEY-----



#### Configure connection to your enterprise directory

Open your copy of the connector-ldap.yml file in a plain-text
editor, and set these values to enable access to your enterprise
directory system:

```
username: "username-goes-here"
password: "password-goes-here"
host: "FQDN.of.host"
base_dn: "base_dn.of.directory"
```

### Configuration options

The main configuration file, user-sync-config.yml, is divided
into several main sections: **adobe_users**, **directory_users**,
**limits**, and **logging**.

- The **adobe_users** section specifies how the User Sync tool
connects to the Adobe Admin Console through the User Management
API. It should point to the separate, secure configuration file
that stores the access credentials.  This is set in the umapi field of
the connectors field.
    - The adobe_users section also can contain exclude_identity_types, 
exclude_adobe_groups, and exclude_users which limit the scope of users
affected by User Sync.  See the later section
[Protecting Specific Accounts from User Sync Deletion](#protecting-specific-accounts-from-user-sync-deletion)
which describes this more fully.
- The **directory_users** subsection contains two subsections,
connectors and groups:
    - The **connectors** subsection points to the separate,
secure configuration file that stores the access credentials for
your enterprise directory.
    - The **groups** section defines the mapping between your
directory groups and Adobe product configurations and user
groups.
    - **directory_users** can also contain keys that set the default country
code and identity type.  See the example configuration files for details.
- The **limits** section sets the `max_adobe_only_users` value that 
prevents User Sync from updating or deleting Adobe user accounts if 
there are more than the specified value of accounts that appear in 
the Adobe organization but not in the directory. This
limit prevents removal of a large number of accounts
in case of misconfiguration or other errors.  This is a required item.
- The **logging** section specifies an audit trail path and
controls how much information is written to the log.

#### Configure connection files

The main User Sync configuration file contains only the names of
the connection configuration files that actually contain the
connection credentials. This isolates the sensitive information,
allowing you to secure the files and limit access to them.

Provide pointers to the connection configuration files in the
**adobe_users** and **directory_users** sections:

```
adobe_users:
  connectors:
    umapi: connector-umapi.yml

directory_users:
  connectors:
    ldap: connector-ldap.yml
```

#### Configure group mapping

Before you can synchronize user groups and entitlements, you must
create user groups and product configurations in the
Adobe Admin Console, and corresponding groups in your enterprise
directory, as described above in
[Set up product-access synchronization](#set-up-product-access-synchronization).

**NOTE:** All groups must exist and have the specified names on
both sides. User Sync does not create any groups on either side;
if a named group is not found, User Sync logs an error.

The **groups** section under **directory_users** must have an entry for
each enterprise directory group that represents access to an
Adobe product or products. For each group entry, list the product
configurations to which users in that group are granted
access. For example:

```YAML
groups:
  - directory_group: Acrobat
    adobe_groups:
      - "Default Acrobat Pro DC configuration"
  - directory_group: Photoshop
    adobe_groups:
      - "Default Photoshop CC - 100 GB configuration"
      - "Default All Apps plan - 100 GB configuration"
```

Directory groups can be mapped to either *product configurations*
or *user groups*. An `adobe_groups` entry can name either kind
of group.

For example:

```YAML
groups:
  - directory_group: Acrobat
    adobe_groups:
      - Default Acrobat Pro DC configuration
  - directory_group: Acrobat_Accounting
    adobe_groups:
      - Accounting_Department
```

#### Configure limits

User accounts are removed from the Adobe system when
corresponding users are not present in the directory and the tool
is invoked with one of the options

- `--adobe-only-user-action delete`
- `--adobe-only-user-action remove`
- `--adobe-only-user-action remove-adobe-groups`

If your organization has a large number of users in the
enterprise directory and the number of users read during a sync
is suddenly small, this could indicate a misconfiguration or
error situation.  The value of `max_adobe_only_users` is a threshold
which causes User Sync to suspend deletion and update of existing Adobe accounts
and report an error if there are
this many fewer users in the enterprise directory (as filtered by query parameters) than in the
Adobe admin console.

Raise this value if you expect the number of users to drop by
more than the current value.

For example:

```YAML
limits:
  max_adobe_only_users: 200
```

This configuration causes User Sync to check if more than
200 user accounts present in Adobe are not found in the enterprise directory (as filtered),
and if so no existing Adobe accounts are updated and an error message is logged.

####  Configure logging

Log entries are written to the console from which the tool was
invoked, and optionally to a log file. A new
entry with a date-time stamp is written to the log each time User Sync
runs.

The **logging** section lets you enable and
disable logging to a file, and controls how much information is
written to the log and console output.

```YAML
logging:
  log_to_file: True | False
  file_log_directory: "path to log folder"
  file_log_level: debug | info | warning | error | critical
  console_log_level: debug | info | warning | error | critical
```

The log_to_file value turns file-logging on or off. Log messages are always
written to the console regardless of the log_to_file setting.

When file-logging is enabled, the file_log_directory value is
required. It specifies the folder where the log entries are to be
written.

- Provide an absolute path or a path relative to the folder
containing this configuration file.
- Ensure that the file and folder have appropriate read/write
permissions.

Log-level values determine how much information is written to the
log file or console.

- The lowest level, debug, writes the most information, and the
highest level, critical, writes the least.
- You can define different log-level values for the file and
console.

Log entries that contain WARNING, ERROR or CRITICAL include a
description that accompanies the status. For example:

> `2017-01-19 12:54:04 7516 WARNING
console.trustee.org1.action - Error requestID: action_5 code:
"error.user.not_found" message: "No valid users were found in the
request"`

In this example, a warning was logged on 2017-01-19 at 12:54:04
during execution. An action caused an error with the code
“error.user.not_found”. The description associated with that
error code is included.

You can use the requestID value to search for the exact request
associated with a reported error. For the example, searching for
“action_5” returns the following detail:

> `2017-01-19 12:54:04 7516 INFO console.trustee.org1.action -
Added action: {"do":
\[{"add": {"product": \["default adobe enterprise support program configuration"\]}}\],
"requestID": "action_5", "user": "cceuser2@ensemble.ca"}`

This gives you more information about the action that resulted in
the warning message. In this case, User Sync attempted to add the
“default adobe enterprise support program configuration” to the
user "cceuser2@ensemble.ca". The add action failed because the
user was not found.

### Example configurations

These examples show the configuration file structures and
illustrate possible configuration values.

##### user-sync-config.yml

```YAML
adobe_users:
  connectors:
    umapi: connector-umapi.yml
  exclude_identity_types:
    - adobeID

directory_users:
  user_identity_type: federatedID
  default_country_code: US
  connectors:
    ldap: connector-ldap.yml
  groups:
    - directory_group: Acrobat
      adobe_groups:
        - Default Acrobat Pro DC configuration
    - directory_group: Photoshop
      adobe_groups:
        - "Default Photoshop CC - 100 GB configuration"
        - "Default All Apps plan - 100 GB configuration"
        - "Default Adobe Document Cloud for enterprise configuration"
        - "Default Adobe Enterprise Support Program configuration"

limits:
  max_adobe_only_users: 200

logging:
  log_to_file: True
  file_log_directory: userSyncLog
  file_log_level: debug
  console_log_level: debug
```

##### connector-ldap.yml

```YAML
username: "LDAP_username"
password: "LDAP_password"
host: "ldap://LDAP_ host"
base_dn: "base_DN"

group_filter_format: "(&(objectClass=posixGroup)(cn={group}))"
all_users_filter: "(&(objectClass=person)(objectClass=top))"
```

##### connector-umapi.yml

```YAML
server:
  # This section describes the location of the servers used for the Adobe user management. Default is:
  # host: usermanagement.adobe.io
  # endpoint: /v2/usermanagement
  # ims_host: ims-na1.adobelogin.com
  # ims_endpoint_jwt: /ims/exchange/jwt

enterprise:
  org_id: "Org ID goes here"
  api_key: "API key goes here"
  client_secret: "Client secret goes here"
  tech_acct: "Tech account ID goes here"
  priv_key_path: "Path to private.key goes here"
  # priv_key_data: "actual key data goes here" # This is an alternative to priv_key_path
```

### Testing your configuration

Use these test cases to ensure that your configuration is working
correctly, and that the product configurations are correctly
mapped to your enterprise directory security groups . Run the
tool in test mode first (by supplying the -t parameter), so that
you can see the result before running live.

#####  User Creation

1. Create one or more test users in enterprise directory.

2. Add users to one or more configured directory/security groups.

3. Run User Sync in test mode. (`./user-sync -t --users all --process-groups --adobe-only-user-action exclude`)

3. Run User Sync not in test mode. (`./user-sync --users all --process-groups --adobe-only-user-action exclude`)

4. Check that test users were created in Adobe Admin Console.

##### User Update

1. Modify group membership of one or more test user in the directory.

1. Run User Sync. (`./user-sync -t --users all --process-groups --adobe-only-user-action exclude`)

2. Check that test users in Adobe Admin Console were updated to
reflect new product configuration membership.

#####  User Disable

1. Remove or disable one or more existing test users in your
enterprise directory.

2. Run User Sync. (`./user-sync -t --users all --process-groups --adobe-only-user-action exclude`)

3. Check that users were removed from configured product
configurations in the Adobe Admin Console.

4. Run User Sync to remove the users (`./user-sync -t --users all --process-groups --adobe-only-user-action delete`) Then run without -t.  Caution: check that only the desired user was removed when running with -t.  This run (without -t) will actually delete users.

5. Check that the user accounts are removed from the Adobe Admin Console.

---

## Command Parameters

Once the configuration files are set up, you can run the User
Sync tool on the command line or in a script. To run the tool,
execute the following command in a command shell or from a
script:

`user-sync` \[ _optional parameters_ \]

The tool accepts optional parameters that determine its
specific behavior in various situations.


| Parameters&nbsp;and&nbsp;argument&nbsp;specifications | Description |
|------------------------------|------------------|
| `-h`<br />`--help` | Show this help message and exit.  |
| `-v`<br />`--version` | Show program's version number and exit.  |
| `-t`<br />`--test-mode` | Run API action calls in test mode (does not execute changes). Logs what would have been executed.  |
| `-c` _filename_<br />`--config-filename` _filename_ | The complete path to the main configuration file, absolute or relative to the working folder. Default filename is "user-sync-config.yml" |
| `--users` `all`<br />`--users` `file` _input_path_<br />`--users` `group` _grp1,grp2_<br />`--users` `mapped` | Specify the users to be selected for sync. The default is `all` meaning all users found in the directory. Specifying `file` means to take input user specifications from the CSV file named by the argument. Specifying `group` interprets the argument as a comma-separated list of groups in the enterprise directory, and only users in those groups are selected. Specifying `mapped` is the same as specifying `group` with all groups listed in the group mapping in the configuration file. This is a very common case where just the users in mapped groups are to be synced.|
| `--user-filter` _regex\_pattern_ | Limit the set of users that are examined for syncing to those matching a pattern specified with a regular expression. See the [Python regular expression documentation](https://docs.python.org/2/library/re.html) for information on constructing regular expressions in Python. The user name must completely match the regular expression.|
| `--update-user-info` | When supplied, synchronizes user information. If the information differs between the enterprise directory side and the Adobe side, the Adobe side is updated to match. This includes the firstname and lastname fields. |
| `--process-groups` | When supplied, synchronizes group membership information. If the membership in mapped groups differs between the enterprise directory side and the Adobe side, the group membership is updated on the Adobe side to match. This includes removal of group membership for Adobe users not listed in the directory side (unless the `--adobe-only-user-action exclude` option is also selected).|
| `--adobe-only-user-action preserve`<br />`--adobe-only-user-action remove-adobe-groups`<br />`--adobe-only-user-action  remove`<br />`--adobe-only-user-action delete`<br /><br/>`--adobe-only-user-action  write-file`&nbsp;filename<br/><br/>`--adobe-only-user-action  exclude` | When supplied, if user accounts are found on the Adobe side that are not in the directory, take the indicated action.  <br/><br/>`preserve`: no action concerning account deletion is taken. This is the default.  There may still be group membership changes if the `--process-groups` option was specified.<br/><br/>`remove-adobe-groups`: The account is removed from user groups and product configurations, freeing any licenses it held, but is left as an active account in the organization.<br><br/>`remove`: In addition to remove-adobe-groups, the account is also removed from the organization, but the user account, with its associated assets, is left in the domain and can be re-added to the organization if desired.<br/><br/>`delete`: In addition to the action for remove, the account is deleted if its domain is owned by the organization.<br/><br/>`write-file`: No action concerning account deletion is taken. The list of user accounts present on the Adobe side but not in the directory is written to the file indicated.  You can then pass this file to the `--adobe-only-user-list` argument in a subsequent run.  There may still be group membership changes if the `--process-groups` option was specified.<br/><br/>`exclude`: No update of any kind is applied to users found only on the Adobe side.  This is used when doing updates of specific users via a file (--users file f) where only users needing explicit updates are listed in the file and all other users should be left alone.<br/><br>Only permitted actions will be applied.  Accounts of type adobeID are owned by the user so the delete action will do the equivalent of remove.  The same is true of Adobe accounts owned by other organizations. |
| `adobe-only-user-list` _filename_ | Specifies a file from which a list of users will be read.  This list is used as the definitive list of "Adobe only" user accounts to be acted upon.  One of the `--adobe-only-user-action` directives must also be specified and its action will be applied to user accounts in the list.  The `--users` option is disallowed if this option is present: only account removal actions can be processed.  |
{: .bordertablestyle }

---

## Usage Scenarios

There are various ways to integrate the User Sync tool into
your enterprise processes, such as:

* **Update users and group memberships.** Sync users and group
memberships by adding, updating, and removing users in Adobe User
Management system.  This is the most general and common use case.
* **Sync only user information.** Use this approach if product
access is to be handled using the Admin Console.
* **Filter users to sync.** You can choose to limit user-information
sync to users in given groups, or limit sync to users that match
a given pattern. You can also sync against a CSV file rather than
a directory system.
* **Update users and group memberships, but handle removals
separately.** Sync users and group memberships by adding and
updating users, but do not remove users in the initial
call. Instead keep a list of users to be removed, then perform
the removals in a separate call.

This section provides detailed instructions for each of these scenarios.

### Update users and group memberships

This is the most typical and common type of invocation. User Sync
finds all changes to user information and to group membership information 
on the enterprise
side. It syncs the Adobe side by adding, updating, and removing
users and user group and product configuration memberships.

By default, only users whose identity type is Enterprise ID or
Federated ID will be created, removed, or have their group
memberships managed by User Sync, because generally Adobe ID
users are not managed in the directory. See the
[description below](#managing-users-with-adobe-ids) under
[Advanced Configuration](#advanced-configuration) if this is how
your organization works.

This example assumes that the configuration file,
user-sync-config.yml, contains a mapping from a directory group
to an Adobe product configuration named **Default Acrobat Pro DC
configuration**.

#### Command

This invocation supplies both the users and process-groups
parameters, and allows user removal with the `adobe-only-user-action remove`
parameter.

```sh
./user-sync –c user-sync-config.yml --users all --process-groups --adobe-only-user-action remove
```

#### Log output during operation

```text
2017-01-20 16:51:02 6840 INFO main - ========== Start Run ==========
2017-01-20 16:51:04 6840 INFO processor - ---------- Start Load from Directory -----------------------
2017-01-20 16:51:04 6840 INFO connector.ldap - Loading users...
2017-01-20 16:51:04 6840 INFO connector.ldap - Total users loaded: 4
2017-01-20 16:51:04 6840 INFO processor - ---------- End Load from Directory (Total time: 0:00:00) ---
2017-01-20 16:51:04 6840 INFO processor - ---------- Start Sync Dashboard ----------------------------
2017-01-20 16:51:05 6840 INFO processor - Adding user with user key: fed_ccewin4@ensemble-systems.com 2017-01-20 16:51:05 6840 INFO dashboard.owning.action - Added action: {"do": \[{"createFederatedID": {"lastname": "004", "country": "CA", "email": "fed_ccewin4@ensemble-systems.com", "firstname": "!Fed_CCE_win", "option": "ignoreIfAlreadyExists"}}, {"add": {"product": \["default acrobat pro dc configuration"\]}}\], "requestID": "action_5", "user": "fed_ccewin4@ensemble-systems.com"}
2017-01-20 16:51:05 6840 INFO processor - Syncing trustee org1... /v2/usermanagement/action/82C654BDB41957F64243BA308@AdobeOrg HTTP/1.1" 200 77
2017-01-20 16:51:07 6840 INFO processor - ---------- End Sync Dashboard (Total time: 0:00:03) --------
2017-01-20 16:51:07 6840 INFO main - ========== End Run (Total time: 0:00:05) ==========
```

#### View result

When the synchronization succeeds, the Adobe Admin Console is
updated.  After this command is executed, your user list and 
product configuration user list in the
Admin Console shows that a user with a Federated identity has
been added to the “Default Acrobat Pro DC configuration.”

![Figure 3: Admin Console Screenshot](media/edit-product-config.png)

#### Sync only users

If you supply only the `users` parameter to the command, the action
finds changes to user information in the enterprise directory and
updates the Adobe side with those changes. You can supply
arguments to the `users` parameter that control which users to look
at on the enterprise side.

This invocation does not look for or update any changes in group
membership. If you use the tool in this way, it is expected you
will control access to Adobe products by updating user group and
product configuration memberships in the Adobe Admin Console.

It also ignores users that are on the Adobe side but no
longer on the directory side, and does not perform any product
configuration or user group management.

```sh
./user-sync –c user-sync-config.yml --users all
```

#### Filter users to sync

Whether or not you choose to sync group membership information,
you can supply arguments to the users parameter that filter which
users are considered on the enterprise directory side, or that
get user information from a CSV file instead of directly from the
enterprise LDAP directory.

#### Sync only users in given groups

This action only looks for changes to user information for users
in the specified groups from. It does not look at any other users
in the enterprise directory, and does not perform any product
configuration or user group management.

```sh
./user-sync –c user-sync-config.yml --users groups "group1, group2, group3"
```

#### Sync only users in mapped groups

This action is the same as specifying `--users groups "..."`, where `...` is all
the groups in the group mapping in the configuration file.

```sh
./user-sync –c user-sync-config.yml --users mapped
```

#### Sync only matching users

This action only looks for changes to user information for users
whose user ID matches a pattern. The pattern is specified with a
Python regular expression.  In this example, we also update group
memberships.

```sh
user-sync --users all --user-filter 'bill@forxampl.com' --process-groups
user-sync --users all --user-filter 'b.*@forxampl.com' --process-groups
```

#### Sync from a file

This action syncs to user information supplied from a CSV file,
instead of looking at the enterprise directory. An example of
such a file, users-file.csv, is provided in the example configuration files
download in `examples/csv inputs - user and remove lists/`.

```sh
./user-sync --users file user_list.csv
```

Syncing from a file can be used in two situations.  First, Adobe users can be managed 
using a spreadsheet.  The spreadsheet lists users, the groups they are in, and
information about them.  Second, if the enterprise directory can provide push notifications
for updates, these notifications can be placed in a csv file and used to drive
User Sync updates.  See the section below for more about this usage scenario.

#### Update users and group memberships, but handle removals separately

If you do not supply the `--adobe-only-user-action` parameter,
you can sync user and group memberships without removing any
users from the Adobe side.

If you want to handle removals separately, you can instruct
the tool to flag users that no longer exist in the enterprise
directory but still exist on the Adobe side. The
`--adobe-only-user-action write-file exiting-users.csv` parameter 
writes out the list of user who
are flagged for removal to a CSV file.

To perform the removals in a separate call, you can pass the
file generated by the `--adobe-only-user-action write-file` parameter, or you
can pass a CSV file of users that you have generated by some
other means. An example of such a file, `3 remove-list.csv`,
is provided in the example-configurations.tar.gz file in the `csv inputs - user and remove lists` folder.

##### Add users and generate a list of users to remove

This action synchronizes all users and also generates a list of users that no longer
exist in the directory but still exist on the Adobe
side.

```sh
./user-sync --users all --adobe-only-user-action write-file users-to-remove.csv
```

##### Remove users from separate list

This action takes a CSV file containing a list of users that have
been flagged for removal, and removes those users from the organization
on the Adobe side. The CSV file is typically the one generated by a
previous call that used the `--adobe-only-user-action write-file` parameter.

You can create a CSV file of users to remove by some other means.
However, if your list includes any users that still exist in your
directory, those users will be added back in on the
Adobe side by the next sync action that adds users.

```sh
./user-sync --adobe-only-user-list users-to-remove.csv --adobe-only-user-action remove
```

#### Delete users that exist on the Adobe side but not in the directory

This invocation supplies both the users and process-groups
parameters, and allows user account deletion with the
adobe-only-user-action delete parameter.

```sh
./user-sync --users all --process-groups --adobe-only-user-action delete
```

#### Delete users from separate list

Similar to the remove users example above, this one deletes users who only exist on the Adobe side, based
on the list generated in a prior run of User Sync.

```sh
./user-sync --adobe-only-user-list users-to-delete.csv --adobe-only-user-action delete
```

### Handling Push Notifications

If your directory system can generate notifications of updates you can use User Sync to
process those updates incrementally.  The technique shown in this section can also be 
used to process immediate updates where an administrator has updated a user or group of 
users and wants to push just those updates immediately into Adobe's user management 
system.  Some scripting may be required to transform the information coming from the 
push notification to a csv format suitable for input to User Sync, and to separate 
deletions from other updates, which mush be handled separately in User Sync.

Create a file, say, `updated_users.csv` with the user update format illustrated in 
the `users-file.csv` example file in the folder `csv inputs - user and remove lists`.  
This is a basic csv file with columns for firstname, lastname, and so on.

    firstname,lastname,email,country,groups,type,username,domain
    John,Smith,jsmith@example.com,US,"AdobeCC-All",enterpriseID
    Jane,Doe,jdoe@example.com,US,"AdobeCC-All",federatedID
 
This file is then provided to User Sync:

```sh
./user-sync --users file updated-users.csv --process-groups --update-users --adobe-only-user-action exclude
```

The --adobe-only-user-action exclude causes User Sync to update only users that are in the updated-users.csv file and to ignore all others.

Deletions are handled similarly.  Create a file `deleted-users.csv` based on the format of `remove-list.csv` in the same example folder and run User Sync:

```sh
./user-sync --adobe-only-user-list deleted-users.csv --adobe-only-user-action remove
```

This will handle deletions based on the notification and no other actions will be taken.  Note that `remove` could be replaced with one of the other actions based on how you want to handle deleted users.

### Action Summary

At the end of the invocation, an action summary will be printed to the log (if the level is INFO or DEBUG). 
The summary provides statistics accumulated during the run. 
The statistics collected include:

- **Total number of Adobe users:** The total number of Adobe users in your admin console
- **Number of Adobe users excluded:** Number of Adobe users that were excluded from operations via the exclude_parameters
- **Total number of directory users:** The total number of users read from LDAP or CSV file
- **Number of directory users selected:** Number of directory users selected via user-filter parameter
- **Number of Adobe users created:** Number of Adobe users created during this run
- **Number of Adobe users updated:** Number of Adobe users updated during this run
- **Number of Adobe users removed:** Number of Adobe users removed from the organization on the Adobe side
- **Number of Adobe users deleted:** Number of Adobe users removed from the organization and Enterprise/Federated user accounts deleted on Adobe side
- **Number of Adobe users with updated groups:** Number of Adobe users that are added to one or more user groups
- **Number of Adobe users removed from mapped groups:** Number of Adobe users that are removed from one or more user groups
- **Number of Adobe users with no changes:** Number of Adobe users that were unchanged during this run

#### Sample action summary output to the log
```text
2017-03-22 21:37:44 21787 INFO processor - ------------- Action Summary -------------
2017-03-22 21:37:44 21787 INFO processor -   Total number of Adobe users: 50
2017-03-22 21:37:44 21787 INFO processor -   Number of Adobe users excluded: 0
2017-03-22 21:37:44 21787 INFO processor -   Total number of directory users: 10
2017-03-22 21:37:44 21787 INFO processor -   Number of directory users selected: 10
2017-03-22 21:37:44 21787 INFO processor -   Number of Adobe users created: 7
2017-03-22 21:37:44 21787 INFO processor -   Number of Adobe users updated: 1
2017-03-22 21:37:44 21787 INFO processor -   Number of Adobe users removed: 1
2017-03-22 21:37:44 21787 INFO processor -   Number of Adobe users deleted: 0
2017-03-22 21:37:44 21787 INFO processor -   Number of Adobe users with updated groups: 2
2017-03-22 21:37:44 21787 INFO processor -   Number of Adobe users removed from mapped groups: 5
2017-03-22 21:37:44 21787 INFO processor -   Number of Adobe users with no changes: 48
2017-03-22 21:37:44 21787 INFO processor - ------------------------------------------
```

---

## Advanced Configuration

User Sync requires additional configuration to synchronize user
data in environments with more complex data structuring.

- When you manage your Adobe ID users out of spreadsheets or your
enterprise directory, you can configure the tool not to ignore
them.
- When your enterprise includes several Adobe organizations, you
can configure the tool to add users in your organization to
groups defined in other organizations.
- When your enterprise user data includes customized attributes
and mappings, you must configure the tool to be able to recognize
those customizations.
- When you want to use username (rather than email) based logins.
- When you want to manage some user accounts manually through the Adobe Admin Console in addition to using User Sync

### Managing Users with Adobe IDs

There is a configuration option `exclude_identity_types` (in 
the `adobe_users` section of the main config file) which
is set by default to ignore Adobe ID users.  If you want User Sync to 
manage some Adobe Id type users, you must turn this option off in the 
config file by removing the `adobeID` entry from under `exclude_identity_types`.

You will probably want to set up a separate sync
job specifically for those users, possibly using CSV inputs
rather than taking inputs from your enterprise directory. If you do this, be sure
to configure this sync job to ignore Enterprise ID and Federated
ID users, or those users are likely to be removed from the
directory!

Removal of Adobe ID users via User Sync may not have the effect you desire:

* If you specify that adobeID users should be
removed from your organization, you will have to re-invite them
(and have them re-accept) if you ever want to add them back in.
* System administrators often use Adobe IDs, so removing Adobe ID
users may inadvertently remove system administrators (including
yourself)

A better practice, when managing Adobe ID users, is simply add
them and manage their group memberships, but never to remove
them.  By managing their group memberships you can disable their
entitlements without the need for a new invitation if you later
want to turn them back on.

Remember that Adobe Id accounts are owned by the end user and 
cannot be deleted.  If you apply a delete action, User Sync will 
automatically substitute the remove action for the delete action.

You can also protect specific Adobe Id users from removal by User Sync 
by using the other exclude configuration items.  See
[Protecting Specific Accounts from User Sync Deletion](#Protecting-Specific-Accounts-from-User-Sync-Deletion)
for more information.

### Accessing Users in Other Organizations

A large enterprise can include multiple Adobe organizations. For
example, suppose a company, Geometrixx, has multiple departments,
each of which has its own unique organization ID and its own
Admin Console.

If an organization uses either Enterprise or Federated user IDs,
it must claim a domain. In a smaller enterprise, the single
organization would claim the domain **geometrixx.com**. However,
a domain can be claimed by only one organization. If multiple
organizations belong to the same enterprise, some or all of them
will want to include users that belong to the enterprise domain.

In this case, the system administrator for each of these
departments would want to claim this domain for identity use. The
Adobe Admin Console prevents multiple departments from claiming
the same domain.  However, once claimed by a single department,
other departments can request access to another department's
domain. The first department to claim the domain is the *owner*
of that domain. That department is responsible for approving
any requests for access by other departments, who are then able
to access users in the domain without any special configuration
requirements.

No special configuration is required to access users in a domain
that you have been granted access to. However, if you want to add
users to user groups or product configurations that are defined
in other organizations, you must configure User Sync so that it
can access those organizations. The tool must be able to find the
credentials of the organization that defines the groups, and be
able to identify groups as belonging to an external organization.


### Accessing Groups in Other Organizations

To configure for access to groups in other organizations, you
must:

- Include additional umapi connection configuration files.
- Tell User Sync how to access these files.
- Identify the groups that are defined in another organization.

##### 1. Include additional configuration files

For each additional organization to which you require access, you
must add a configuration file that provides the access
credentials for that organization. The file has the
same format as the connector-umapi.yml file.  Each additional organization will be referred to by a short nickname (that you define).  You can name the configuration file that has the access credentials for that organization however you like.  

For example, suppose the additional organization is named "department 37".  The config file for it might be named: 

`department37-config.yml`

##### 2. Configure User Sync to access the additional files


The `adobe-users` section of the main configuration file must
include entries that reference these files, and
associate each one with the short organization name. For
example:

```YAML
adobe-users:
  connectors:
    umapi:
      - connector-umapi.yml
      - org1: org1-config.yml
      - org2: org2-config.yml
      - d37: department37-config.yml  # d37 is short name for example above
```

If unqualified file names are used, the configuration files must be in the same folder as the main configuration file that references them.

Note that, like your own connection
configuration file, they contain sensitive information that must
be protected.

##### 3. Identify groups defined externally

When you specify your group mappings, you can map an enterprise
directory group to an Adobe user group or product configuration defined in another
organization.

To do this, use the organization identifier as a prefix to the
group name. Join them with "::". For example:

```YAML
- directory_group: CCE Trustee Group
  adobe_groups:
    - "org1::Default Adobe Enterprise Support Program configuration"
    - "d37::Special Ops Group"
```

### Custom Attributes and Mappings

It is possible to define custom mappings of directory attribute
or other values to the fields used to define and update users:
first name, last name, email address, user name, country, and group membership.
Normally, standard attributes in the directory are used to
obtain these values.  You can define other attributes to be used and
specify how field values should be computed.

To do this, you must configure User Sync to recognize any non-standard
mappings between your enterprise directory user data and Adobe
user data.  Non-standard mappings include:

- Values for user name, groups, country, or email that are in or
are based on any non-standard attribute in the directory.
- Values for user name, groups, country, or email must be
computed from directory information.
- Additional user groups or products that must be added to or
removed from the list for some or all users.

Your configuration file must specify any custom attributes to be
fetched from the directory. In addition, you must specify any
custom mapping for those attributes, and any computation or
action to be taken to sync the values. The custom action is
specified using a small block of Python code. Examples and
standard blocks are provided.

The configuration for custom attributes and mappings go in a separate
configuration file.  That file is referenced from the main
configuration file in the `directory_users` section:

```
directory_users:
  extension: extenstions_config.yml  # reference to file with custom mapping information
```

Custom attribute handling is performed for each user, so the
customizations are configured in the per-user subsection of the
extensions section of the main User Sync configuration file.

```
extensions:
  - context: per_user
    extended_attributes:
      - my-attribute-1
      - my-attribute-2
    extended_adobe_groups:
      - my-adobe-group-1
      - my-adobe-group-2
    after_mapping_hook: |
        pass # custom python code goes here
```

#### Adding custom attributes

By default, User Sync captures these standard attributes for each
user from the enterprise directory system:

* `givenName` - used for Adobe-side first name in profile
* `sn` - used for Adobe-side last name in profile
* `c` - used for Adobe-side country (two-letter country code)
* `mail` - used for Adobe-side email
* `user` - used for Adobe-side username only if doing Federated ID via username

In addition, User Sync captures any attribute names that appear in
filters in the LDAP connector configuration.

You can add attributes to this set by specifying them in an
`extended_attributes` key in the main configuration file, as shown above. The
value of the `extended_attributes` key is a YAML list of strings, with
each string giving the name of a user attribute to be
captured. For example:

```YAML
extensions:
  - context: per-user
    extended_attributes:
    - bc
    - subco
```

This example directs User Sync to capture the `bc` and `subco`
attributes for every user loaded.

If one or more of the specified attributes is missing from the
directory information for a user, those attributes are
ignored. Code references to such attributes will return the
Python `None` value, which is normal and not an error.

#### Adding custom mappings

Custom mapping code is configured using an extensions section in
the main (user sync) config file. Within extensions, a per-user
section will govern custom code that's invoked once per user.

The specified code would be executed once for each user, after
attributes and group memberships have been retrieved from the
directory system, but before actions to Adobe have been
generated.

```YAML
extensions:
  - context: per-user
    extended_attributes:
      - bc
      - subco
    extended_adobe_groups:
      - Acrobat_Sunday_Special
      - Group for Test 011 TCP
    after_mapping_hook: |
      bc = source_attributes['bc']
      subco = source_attributes['subco']
      if bc is not None:
          target_attributes['country'] = bc[0:2]
          target_groups.add(bc)
      if subco is not None:
          target_groups.add(subco)
      else:
          target_groups.add('Undefined subco')
```

In this example, two custom attributes, bc, and subco, are
fetched for each user that is read from the directory. The custom
code processes the data for each user:

- The country code is taken from the first 2 characters in the bc
attribute.

    This shows how you can use custom directory attributes to provide
values for standard fields being sent to Adobe.

- The user is added to groups that come from subco attribute and
the bc attribute (in addition to any mapped groups from the group
map in the configuration file).

    This shows how to customize the group or product configuration
list to get users synced into additional groups.

If the hook code references Adobe groups or product
configurations that do not already appear in the **groups**
section of the main configuration file, they are listed under
**extended_adobe_groups**. This list effectively extends the
set of Adobe groups that are considered . See
[Advanced Group and Product Management](#advanced-group-and-product-management)
for more information.

#### Hook code variables

The code in the `after_mapping_hook` is isolated from the rest of
the User Sync program except for the following variables.

##### Input values

The following variables can be read in the custom code.  They
should not be written, and writes tot them have no effect; they
exist to express the source directory data about the user.

* `source_attributes`: A per-user dictionary of user attributes
  retrieved from the directory system. As a Python dictionary,
  technically, this value is mutable, but changing it from custom
  code has no effect.

* `source_groups`: A frozen set of directory groups found for a
specific user while traversing configured directory groups.

##### Input/output values

The following variables can be read and written by the custom
code. They come in carrying data set by the default attribute and
group mapping operations on the current directory user, and can
be written so as to change the actions performed on the
corresponding Adobe user.

* `target_attributes`: A per-user Python dictionary whose keys
are the Adobe-side attributes that are to be set. Changing a
value in this dictionary will change the value written on the
Adobe side. Because Adobe pre-defines a fixed set of attributes,
adding a key to this dictionary has no effect.  The keys in this
dictionary are:
    * `firstName` - ignored for AdobeID, used elsewhere
    * `lastName` - ignored for AdobeID, used elsewhere
    * `email` - used everywhere
    * `country` - ignored for AdobeID, used elsewhere
    * `username` - ignored for all but Federated ID
      [configured with username-based login](https://helpx.adobe.com/enterprise/help/configure-sso.html)
    * `domain` - ignored for all but Federated ID [configured with username-based login](https://helpx.adobe.com/enterprise/help/configure-sso.html)
* `target_groups`: A per-user Python set that collects the
Adobe-side user groups and product configurations to which the
user is added when `process-groups` is specified for the sync
run.  Each value is a set of names. The set is initialized by
applying the group mappings in the main configurations file, and
changes made to this set (additions or removals) will change the
set of groups that are applied to the user on the Adobe side.
* `hook_storage`: A per-user Python dictionary that is empty the
first time it is passed to custom code, and persists across
calls.  Custom code can store any private data in this
dictionary. If you use external script files, this is a suitable
place to store the code objects created by compiling these files.
* `logger`: An object of type `logging.logger` which outputs to
the console and/or file log (as per the logging configuration).

### Advanced Group and Product Management

The **group** section of the main configuration file defines a
mapping of directory groups to Adobe user groups and product
configurations.

- On the enterprise directory side, User Sync selects a set of
users from your enterprise directory, based on the LDAP query,
the `users` command line parameter, and the user filter, and
examines these users to see if they are in any of the mapped
directory groups. If they are, User Sync uses the group map to
determine which Adobe groups those users should be added to.
- On the Adobe side, User Sync examines the membership of mapped
groups and product configurations. If any user in those groups is
_not_ in the set of selected directory users, User Sync removes
that user from the group. This is usually the desired behavior
because, for example, if a user is in the Adobe Photoshop product
configuration and they are removed from the enterprise directory,
you would expect them to be removed from the group so that they
are no longer allocated a license.

![Figure 4: Group Mapping Example](media/group-mapping.png)

This workflow can present difficulties if you want to divide the
sync process into multiple runs in order to reduce the number of
directory users queried at once. For example, you could do a run
for users beginning with A-M and another with users N-Z. When you
do this, each run must target different Adobe user groups and
product configurations.  Otherwise, the run for A-M would remove
users from mapped groups who are in the N-Z set.

To configure for this case, use the Admin Console to create user
groups for each user subset (for example, **photoshop_A_M** and
**photoshop_N_Z**), and add each of the user groups separately to
the product configuration (for example, **photoshop_config**). In
your User Sync configuration, you then map only the user groups,
not the product configurations. Each sync job targets one user
group in its group map.  It updates membership in the user group,
which indirectly updates the membership in the product
configuration.

### Removing Group Mappings

There is potential confusion when removing a mapped group. Say a 
directory group `acrobat_users` is mapped to the Adobe group `Acrobat`. 
and you no longer want to map the group to `Acrobat` so you take out 
the entry. The result is that all of the users are left in the 
`Acrobat` group because `Acrobat` is no longer a mapped group so user 
sync leaves it alone. It doesn't result in removing all the users 
from `Acrobat` as you might have expected.

If you also wanted the users removed from the `Acrobat` group, you can
manually remove them using the Admin Console, or you can (at least 
temporarily) leave the entry in the group map in the configuration
file, but change the directory group to a name that you know does
not exist in the directory, such as `no_directory_group`.  The next sync 
run will notice that there are users in the Adobe group who are 
not in the directory group and 
they will all be moved.  Once this has happened, you can remove
the entire mapping from the configuration file.

### Working with Username-Based Login

On the Adobe Admin Console, you can configure a federated domain to use email-based user login names or username-based (i.e., non-email-based) login.   Username-based login can be used when email addresses are expected to change often or your organization does not allow email addresses to be used for login.  Ultimately, whether to use username-based login or email-based login depends on a company's overall identity strategy.

To configure User Sync to work with username logins, you need to set several additional configuration items.

In the `connector-ldap.yml` file:

- Set the value of `user_username_format` to a value like '{attrname}' where attrname names the directory attribute whose value is to be used for the user name.
- Set the value of `user_domain_format` to a value like '{attrname}' if the domain name comes from the named directory attribute, or to a fixed string value like 'example.com'.

When processing the directory, User Sync will fill in the username and domain values from those fields (or values).

The values given for these configuration items can be a mix of string characters and one or more attribute names enclosed in curly-braces "{}".  The fixed characters are combined with the attribute value to form the string used in processing the user.

For domains that use username-based login, the `user_username_format` configuration item should not produce an email address; the "@" character is not allowed in usernames used in username-based login.

If you are using username-based login, you must still provide a unique email address for every user, and that email address must be in a domain that the organization has claimed and owns. User Sync will not add a user to the Adobe organization without an email address.

### Protecting Specific Accounts from User Sync Deletion

If you drive account creation and removal through User Sync, and want to manually create a few accounts, you may need this feature to keep User Sync from deleting the manually created accounts.

In the `adobe_users` section of the main configuration file you can include
the following entries:

```YAML
adobe_users:
  exclude_adobe_groups: 
      - special_users       # Adobe accounts in the named group will not be removed or changed by user sync
  exclude_users:
      - ".*@example.com"    # users whose name matches the pattern will be preserved by user sync 
      - another@example.com # can have more than one pattern
  exclude_identity_types:
      - adobeID             # causes user sync to not remove accounts that are AdobeIds
      - enterpriseID
      - federatedID         # you wouldn’t have all of these since that would exclude everyone  
```

These are optional configuration items.  They identify individual or groups
of accounts and the identified accounts are protected from deletion by 
User Sync.  These accounts may still be added to or removed from user
groups or Product Configurations based on the group map entries and
the `--process-groups` command line option.  

If you want to prevent User Sync from removing these accounts from groups, only place them in groups not under control of User Sync, that is, in groups 
that are not named in the group map in the config file.

- `exclude_adobe_groups`: The values of this configuration item is a list of strings that name Adobe user groups or PCs.  Any users in any of these groups are preserved and never deleted as Adobe-only users.
- `exclude_users`: The values of this configuration item is a list of strings that are patterns that can match Adobe user names.  Any matching users are preserved and never deleted as Adobe-only users.
- `exclude_identity_types`:  The values of this configuration item is a list of strings that can be "adobeID", "enterpriseID", and "federatedID".  This causes any account that is of the listed type(s) to be preserved and never deleted as Adobe-only users.


### Working With Nested Directory Groups in Active Directory

If your directory groups are structured in a nested manner so that users are 
not in one simple named directory group, you will need to run more complex
LDAP queries to enumerate the list of users.  For example you might have a group
nesting structure like this:


    All_Divisions
		Blue_Division
		       User1@example.com
		       User2@example.com
		Green_Division
		       User3@example.com
		       User4@example.com


To handle this type of nesting structure, in your LDAP config file, 
set the value of group_filter_format as follows:

    group_filter_format: "(memberOf:1.2.840.113556.1.4.1941:=cn={group},cn=users,DC=example,DC=com)"
    
where this part of the query:

    cn={group},cn=users,DC=example,DC=com

is the full distinguished name of the group.  The entry {group} will be replaced by the actual group when User Sync runs the group membership query.  This example assumes the group is located in the 
directory in users.example.com.  If the group is located elsewhere in the directory, you
would need to provide the appropriate path to reference it.  (example.com would be replaced by
your actual domain name also.) 

Once this is set, you can use `All_Divisions` in the group map as the directory group and/or specify `--users group All_Divisions` as the source for users.

How this works is explained in the accepted answer to this [StackOverflow](http://stackoverflow.com/questions/6195812/ldap-nested-group-membership) question and in this [MS technote](https://msdn.microsoft.com/en-us/library/aa746475%28VS.85%29.aspx). AD supports the LDAP_MATCHING_RULE_IN_CHAIN predicate which will search the entire ancestry tree to find containment.  1.2.840.113556.1.4.1941 is the precise OID you must specify to use that predicate.

This can be a very expensive filter, and should be used very carefully.

---

## Deployment Best Practices

The User Sync tool is designed to run with limited or no human
interaction, once it is properly configured. You can use a
scheduler in your environment to run the tool with whatever
frequency you need.

- The first few executions of the User Sync Tool can take a long
time, depending on how many users need to be added into the Adobe
Admin Console. We recommend that you run these initial executions
manually, before setting it up to run as a scheduled task, in
order to avoid having multiple instances running.
- Subsequent executions are typically faster, as they only need
to update user data as needed. The frequency with which you
choose to execute User Sync depends on how often your
enterprise directory changes, and how quickly you want the changes
to show up on the Adobe side.
- Running User Sync more often than once every 2 hours is not recommended.

### Security recommendations

Given the nature of the data in the configuration and log files,
a server should be dedicated for this task and locked down with
industry best practices. It is recommended that a server that
sits behind the enterprise firewall be provisioned for this
application. Only privileged users should be able to connect to
this machine. A system service account with restricted privileges
should be created that is specifically intended for running the
application and writing log files to the system.

The application makes GET and POST requests of the User
Management API against a HTTPS endpoint. It constructs JSON data
to represent the changes that need to be written to the Admin
console, and attaches the data in the body of a POST request to
the User Management API.

To protect the availability of the Adobe back-end user identity
systems, the User Management API imposes limits on client access
to the data.  Limits apply to the number of calls that an
individual client can make within a time interval, and global
limits apply to access by all clients within the time period. The
User Sync tool implements back off and retry logic to prevent the
script from continuously hitting the User Management API when it
reaches the rate limit. It is normal to see messages in the
console indicating that the script has paused for a short amount
of time before trying to execute again.

Starting in User Sync 2.1, there are two additional techniques available
for protecting credentials.  The first uses the operating system credential
store to store individual configuration credential values.  The second uses
a mechanism you must provide to store the entire configuration file for umapi
and/or ldap which includes all the credentials required.  These are
detailed in the next two sections.

#### Storing Credentials in OS Level Storage

To setup User Sync to pull credentials from the Python Keyring OS credential store, set the connector-umapi.yml and connector-ldap.yml files as follows:

connector-umapi.yml

	server:
	
	enterprise:
	  org_id: your org id
	  secure_api_key_key: umapi_api_key
	  secure_client_secret_key: umapi_client_secret
	  tech_acct: your tech account@techacct.adobe.com
	  secure_priv_key_data_key: umapi_private_key_data

Note the change of `api_key`, `client_secret`, and `priv_key_path` to `secure_api_key_key`, `secure_client_secret_key`, and `secure_priv_key_data_key`, respectively.  These alternate configuration values give the key names to be looked up in the user keychain (or the equivalent service on other platforms) to retrieve the actual credential values.  In this example, the credential key names are `umapi_api_key`, `umapi_client_secret`, and `umapi_private_key_data`.

The contents of the private key file is used as the value of `umapi_private_key_data` in the credential store.

The credential values will be looked up using the specified key names with the user being the org_id value.


connector-ldap.yml

	username: "your ldap account username"
	secure_password_key: ldap_password 
	host: "ldap://ldap server name"
	base_dn: "DC=domain name,DC=com"

The LDAP access password will be looked up using the specified key name
(`ldap_password` in this example) with the user being the specified username
config value.

Credentials are stored in the underlying operating system secure store.  The specific storage system depends in the operating system.

| OS | Credential Store |
|------------|--------------|
|Windows | Windows Credential Vault |
| Mac OS X | Keychain |
| Linux | Freedesktop Secret Service or KWallet |

On Linux, the secure storage application would have been installed and configured by the OS vendor.

The credentials are added to the OS secure storage and given the username and credential id that you will use to specify the credential.  For umapi credentials, the username is the organization id.  For the LDAP password credential, the username is the LDAP username.  You can pick any identifier you wish for the specific credentials; they must match between what is in the credential store and the name used in the configuration file.  Suggested values for the key names are shown in the examples above.


#### Storing Credential Files in External Management Systems

As an alternative to storing credentials in the local credential store, it is possible to integrate User Sync with some other system or encryption mechanism.  To support such integrations, it is possible to store the entire configuration files for umapi and ldap externally in some other system or format.

This is done by specifying, in the main User Sync configuration file, a command to be executed whose output is used as the umapi or ldap configuration file contents.  You will need to provide the command that fetches the configuration information and sends it to standard output in yaml format, matching what the configuration file would have contained.

To set this up, use the following items in the main configuration file.


user-sync-config.yml (showing partial file only)

	adobe_users:
	   connectors:
	      # umapi: connector-umapi.yml   # instead of this file reference, use:
	      umapi: $(read_umapi_config_from_s3)
	
	directory_users:
	   connectors:
	      # ldap: connector-ldap.yml # instead of this file reference, use:
	      ldap: $(read_ldap_config_from_server)
 
The general format for external command references is

	$(command args)

The above examples assume there is a command with the name `read_umapi_config_from_s3`
and `read_ldap_config_from_server` that you have supplied.

A command shell is launched by User Sync which
runs the command.  The standard output from the command is captured and that
output is used as the umapi or ldap configuration file.

The command is run with the working directory as the directory containing the configuration file.

If the command terminates abnormally, User Sync will terminate with an error.

The command can reference a new or existing program or a script.

Note: If you use this technique for the connector-umapi.yml file, you will want to embed the private key data in connector-umapi-yml directly by using the priv_key_data key and the private key value.  If you use the priv_key_path and the filename containing the private key, you would also need to store the private key somewhere 
secure and have a command that retrieves it in the file reference.

### Scheduled task examples

You can use a scheduler provided by your operating system to run
the User Sync tool periodically, as required by your
enterprise. These examples illustrate how you might configure the
Unix and Windows schedulers.

You may want to set up a command file that runs UserSync with
specific parameters and then extracts a log summary and emails it
to those responsible for monitoring the sync process. These
examples work best with console log level set to INFO

```YAML
logging:
  console_log_level: info
```

#### Run with log analysis in Windows

The following example shows how to set up a batch file `run_sync.bat` in
Windows.

```sh
python C:\\...\\user-sync.pex --users file users-file.csv --process-groups | findstr /I "WARNING ERROR CRITICAL ---- ==== Number" > temp.file.txt
rem email the contents of temp.file.txt to the user sync administration
sendmail -s “Adobe User Sync Report for today” UserSyncAdmins@example.com < temp.file.txt
```

*NOTE*: Although we show use of `sendmail` in this example, there
is no standard email command-line tool in Windows.  Several are
available commercially.

#### Run with log analysis on Unix platforms

The following example shows how to set up a shell file
`run_sync.sh` on Linux or Mac OS X:

```sh
user-sync --users file users-file.csv --process-groups | grep "CRITICAL\|WARNING\|ERROR\|=====\|-----\|number of\|Number of" | mail -s “Adobe User Sync Report for `date +%F-%a`” UserSyncAdmins@example.com
```

#### Schedule a UserSync task

##### Cron

This entry in the Unix crontab will run the User Sync tool at 4
AM each day:

```text
0 4 * * * /path/to/run_sync.sh
```

Cron can also be setup to email results to a specified user or
mailing list. Check the documentation on cron for your system
for more details.

##### Windows Task Scheduler

This command uses the Windows task scheduler to run the User Sync
tool every day starting at 4:00 PM:

```text
schtasks /create /tn "Adobe User Sync" /tr C:\path\to\run_sync.bat /sc DAILY /st 16:00
```

Check the documentation on the windows task scheduler (`help
schtasks`) for more details.

There is also a GUI for managing windows scheduled tasks. You can
find the Task Scheduler in the Windows administrative control
panel.
