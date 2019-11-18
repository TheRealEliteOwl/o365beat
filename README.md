# O365beat

O365beat is an open source log shipper used to fetch Office 365 audit logs from the [Office 365 Management Activity API](https://docs.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-reference) and forward them with all the flexibility and capability provided by the [beats platform](https://github.com/elastic/beats) (specifically, [libbeat](https://github.com/elastic/beats/tree/master/libbeat)).

**The latest release is [v1.4.2](https://github.com/counteractive/o365beat/releases/latest)**.  It:

1. Includes new kibana visualizations and a dashboard since v1.4.1, showing `AlertTriggered` events from Microsoft's Advanced Threat Protection service, a chart of common client IP addresses, a list of unique users, and a running stream of summarized activity.
1. Updates processors to better handle certain log fields.  Specifically, the API provides `Parameters` and `ExtendedProperties` fields as arrays of objects with just `Name` and `Value` keys, which is _very_ confusing and difficult to work with, and causes issues with elasticsearch.  We found this was true of the `ModifiedProperties` field as well.  This version ships all those as strings, which can then be deserialized or parsed with string-based tools.  Most importantly, it stops indexing errors and dropped events.

It closes a number of issues (#12, #13, and #14), but there is still a lot on the [to-do list](#tasks) and probably more than a few bugs still hiding out there! Please open an issue or submit a pull request if you notice any problems in testing or production.

## Getting Started with O365beat

The easiest way to get started with o365beat is to use the pre-built binaries available in the [latest release](https://github.com/counteractive/o365beat/releases/latest/).

These pre-built packages include configuration files which contain all the necessary credential information to connect to the audit logs for your tenancy.  The default configuration file ([`o365beat.yml`](./_meta/beat.yml)) pulls this information from your environment or beats keystores (see [this issue](https://github.com/counteractive/o365beat/issues/11) or the [filebeat docs](https://www.elastic.co/guide/en/beats/filebeat/current/keystore.html)), like so:

```yaml
o365beat:
  # period Defines how often API is polled for new content blobs
  # 5 min default, as new content (probably) isn't published too often
  # period: 5m

  # pull secrets from environment (e.g, > set -a; . ./ENV_FILE; set +a;)
  # or hard-coded here:
  tenant_domain: ${O365BEAT_TENANT_DOMAIN:}
  client_secret: ${O365BEAT_CLIENT_SECRET:}
  client_id:     ${O365BEAT_CLIENT_ID:}     # aka application id (GUID)
  directory_id:  ${O365BEAT_DIRECTORY_ID:}  # aka tenant id (GUID)
  registry_file_path: ${O365BEAT_REGISTRY_PATH:./o365beat.state}

  # the following content types will be pulled from the API
  # for available types, see https://docs.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-reference#working-with-the-office-365-management-activity-api
  content_types:
    - Audit.AzureActiveDirectory
    - Audit.Exchange
    - Audit.SharePoint
    - Audit.General
```

**NOTE: Due to a quirk in the libbeat build system, the default config file contains an additional `processors` section that gets merged into the o365beat.yml and shadows the custom processors used by this beat. You must manually remove the second `processors` section, or merge the two, to avoid problems.**  Please see [this issue](https://github.com/counteractive/o365beat/issues/9) for more information, we're working on a durable fix.

See below for more details on these values.

**NOTE:** If you decide to hard-code these values, be sure to replace the `${:}` syntax, which [pulls from the environment](https://www.elastic.co/guide/en/beats/libbeat/current/config-file-format-env-vars.html).  For example, use `tenant_domain: acme.onmicrosoft.com` *not* `tenant_domain: ${acme.onmicrosoft.com:}`.

### Prerequisites and Permissions

O365beat requires that you [enable audit log search](https://docs.microsoft.com/en-us/microsoft-365/compliance/turn-audit-log-search-on-or-off#turn-on-audit-log-search) for your Office 365 tenancy, done through the Security and Compliance Center in the Office 365 Admin Portal.

It also needs access to the Office 365 Management API: instructions for setting this up are available in the [Microsoft documentation](https://docs.microsoft.com/en-us/office/office-365-management-api/get-started-with-office-365-management-apis#register-your-application-in-azure-ad).

Once you have these set up, you'll be able to get the information needed in the config file.  The naming conventions for the settings are a bit odd, in `o365beat.yml` you’ll see some of the synonyms: client id is also called the application id, and the directory id is also called the tenant id.  In the Azure portal, go to "App registrations" and you’ll see the Application (Client) ID – a GUID – right there in the application list.  If you click on that you’ll see the application (client) id and the directory (tenant) id in the top area.

![App Details in Azure Portal](./docs/o365beat-readme-1.jpg)

The client secret is a little trickier, you can create them by clicking the "Certificates & secrets" link on the left there.  Be sure to copy it somewhere or you’ll have to create a new one … there’s no facility for viewing them later.  The [default config file](./o365beat.yml) expects these config values to be in your environment (i.e., as environment variables), named O365BEAT_TENANT_DOMAIN, O365BEAT_CLIENT_SECRET, etc.  You can hard-code them in that file if you like, especially when testing, just be smart about the permissions.

Finally, the Azure app registration permissions should look like this:

![App Permissions in Azure Portal](./docs/o365beat-readme-2.jpg)

You can edit those using that “API permissions” link on the left, with [more detailed instructions available from Microsoft](https://docs.microsoft.com/en-us/office/office-365-management-api/get-started-with-office-365-management-apis#specify-the-permissions-your-app-requires-to-access-the-office-365-management-apis).  The beat should automatically subscribe you to the right feeds, though that functionality is currently undergoing testing.

### Run

To run O365beat with all debugging output enabled, run:

```bash
./o365beat --path.config . -c o365beat.yml -e -d "*" # add --strict.perms=false under WSL 1
```

State is maintained in the `registry_file_path` location, by default in the working directory as `o365beat-registry.json`.  This file currently contains only a timestamp representing the creation date of the last content blob retrieved, to prevent repeat downloads.

**NOTE:** By default o365beat doesn't know where to look for its configuration so you have to specify that explicitly.  If you see errors authenticating it may be the beat's not seeing your config.  Future versions will have more helpful error messages in this regard.

### Receive with Logstash

If you're receiving o365beat logs with [logstash](https://www.elastic.co/products/logstash), use the input type `beats`:

```ruby
input {
  beats {
    port => "5044"
  }
}
```

### Schema

As of v1.2.0, o365beat includes a [processor](https://github.com/elastic/beats/blob/master/libbeat/docs/processors-using.asciidoc#convert) to map the raw API-provided events to Elastic Common Schema ([ECS](https://www.elastic.co/guide/en/ecs/current/index.html)) fields.  This allows this beat to work with standard Kibana dashboards, including capabilities in [Elastic SIEM](https://www.elastic.co/products/siem).  Updates in v1.4.0 and v1.4.1 corrected some parsing issues and included at least one more ECS field.

Implementing this as a processor means you can disable it if you don't use the ECS functionality, or change from "copy" to "rename" if you _only_ use ECS.  We may end up adding some ECS stuff in the "core" of the beat as well, but this is a decent start.  These processors are critical for the proper functioning of the beat and its visualizations.  Disabling or modifying them can lead to dropped events or other issues.  **Please update with caution.**

See the [Office 365 Management API schema documentation](https://docs.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-schema) for details on the raw events.  The ECS mapping is as follows (excerpt from [`o365beat.yml`](./_meta/beat.yml)):

```yaml
# from: https://docs.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-schema
# to: https://www.elastic.co/guide/en/ecs/current/ecs-client.html

processors:
  - convert:
      fields:
        - {from: Id, to: 'event.id', type: string}                # ecs core
        - {from: RecordType, to: 'event.code', type: string}      # ecs extended
        # - {from: "CreationTime", to: "", type: ""}              # @timestamp
        - {from: Operation, to: 'event.action', type: string}     # ecs core
        - {from: OrganizationId, to: 'cloud.account.id', type: string} # ecs extended
        # - {from: UserType, to: '', type: ''}                    # no ecs mapping
        # - {from: UserKey, to: '', type: ''}                     # no ecs mapping
        - {from: Workload, to: 'event.category', type: string}    # ecs core
        - {from: ResultStatus, to: 'event.outcome', type: string} # ecs extended
        # - {from: ObjectId, to: '', type: ''}                    # no ecs mapping
        - {from: UserId, to: 'user.id', type: string}             # ecs core
        - {from: ClientIP, to: 'client.ip', type: ip}             # ecs core
        - {from: 'dissect.clientip', to: 'client.ip', type: ip}   # ecs core
        # - {from: "Scope", to: "", type: ""}                     # no ecs mapping
        - {from: Severity, to: 'event.severity', type: string}    # ecs core
        # the following fields use the challenging array-of-name-value-pairs format
        # converting them to strings fixes issues in elastic, eases non-script parsing
        # easier to rehydrate into arrays from strings than vice versa:
        - {from: Parameters, type: string}                        # no ecs mapping
        - {from: ExtendedProperties, type: string}                # no ecs mapping
```

Please open an issue or a pull request if you have suggested improvements to this approach.

*If you'd like to build yourself, read on.*

### Build Requirements

* [Golang](https://golang.org/dl/) 1.7

### Build

To build the binary for O365beat run the command below. This will grab vendor dependencies if you don't have them already, and generate a binary in the same directory with the name o365beat.

```bash
make
```

### Test (none so far!)

To test O365beat, run the following command:

```bash
make testsuite
```

alternatively:

```bash
make unit-tests
make system-tests
make integration-tests
make coverage-report
```

The test coverage is reported in the folder `./build/coverage/`

### Update

Each beat has a template for the mapping in elasticsearch and a documentation for the fields
which is automatically generated based on `fields.yml` by running the following command.

```bash
make update
```

### Cleanup

To clean  O365beat source code, run the following command:

```bash
make fmt
```

To clean up the build directory and generated artifacts, run:

```bash
make clean
```

### Clone

To clone O365beat from the git repository, run the following commands:

```bash
mkdir -p ${GOPATH}/src/github.com/counteractive/o365beat
git clone https://github.com/counteractive/o365beat ${GOPATH}/src/github.com/counteractive/o365beat
```

For further development, check out the [beat developer guide](https://www.elastic.co/guide/en/beats/libbeat/current/new-beat.html).

## Packaging

The beat frameworks provides tools to cross-compile and package your beat for different platforms. This requires [docker](https://www.docker.com/) and vendor-ing as described above. To build packages of your beat, run the following command:

```bash
make release
```

Be sure you have python, virtualenv, gcc, and docker installed, and that the user you're using to build the release is in the `docker` group (if not, it'll just hang with no helpful error message).

This will fetch and create all images required for the build process. The whole process to finish can take several minutes.

## Tasks

* [ ] Tests
* [ ] ECS field mappings beyond the API's [common schema](https://docs.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-schema#common-schema)
* [x] Add visualizations and dashboard
* [x] Update underlying libbeat to ~~7.3.x~~ 7.4.x (currently 7.2.x)
* [x] ECS field mappings for API's [common schema](https://docs.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-schema#common-schema)

## Changelog

* v1.4.2 - Fixes multiple processor bugs (closes issues #12, #13, and #14)
* v1.4.1 - Added kibana visualizations and dashboard and updated processors to better handle fields containing data arrays
* v1.4.0 - Bumped libbeat to v7.4.0 and fixed throttling issue
* v1.3.1 - Updated documentation and improved error messages
* v1.3.0 - Fixed auto-subscribe logic and updated documentation
* v1.2.0 - Initial production release
