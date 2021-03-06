# Config Server [PLANNING]

Goals:

- ability to reference config values from a manifest without relying on the client
- ability to generate certain types of properties by default

Currently all solutions involve client side interpolation (eg spiff family, ERB, deployment properties).

Proposal: Introduce optional config API in the Director to fetch property values. Director will evaluate manifests and replace marked values with values fetched via config API. The Director will not store fetched values but rather only keep them in memory. By default BOSH team will include simple config server that will store its values in some database.

## API

Following public API will be used by the Director to contact config server:

- GET /v1/data/&lt;some-key-path>
  - whenever Director needs to retrieve a value it will use GET action
  - { "type": "value", "value": <any json value> }

- PUT /v1/data/&lt;some-key-path>
  - manual config value update
  - { "type": "certificate", "value": <any json value> }

- POST /v1/data/&lt;some-key-path>
  - whenever Director generates a value it will be saved into the config server
  - {
      "type": "value",
      "parameters": { <opaque> }
    }

### Generic value generation parameters

The Director will not include any params when generating generic value.

### Certificate generation parameters

The Director will include following params when generating certificates.

```
{
  type": "certificate",
  parameters": {
    "common_name": "bosh.io",
    "alternative_name": ["blah.bosh.io", "10.10.10.1"],
    "organization": "Blah",
    "organization_unit": "Blah Dev",
    "locality": "San Francisco",
    "state": "CA",
    "country": "US"
  }
}
```

name: { ... }

Values could be any valid JSON object.

### Authentication

Director will use UAA for authentication with the config server. The Director will be configured with a UAA client and secret (similarly to how HM is configured).

## Manifest configuration

When manifest includes "{{key-name}}" directive it will be parsed out by the Director and value associated with "key-name" will be retrieved from the config server. This should happen before manifest is used for determining if instances need an update.

Example how manifest may reference values:

```
instance_groups:
- name: nats
  jobs:
  - name: nats
    release: nats
    properties:
    	password: {{nats.password}}
```

Note that the Director should first parse YAML manifest and only then replace leaf values using curly braces. This does not allow to do something like this: "https://api.{{domain}}".

Client side replacement of "{{...}}" will not be affected.

## Stories

- create a release with a simple config server that has a single GET endpoint for fetching a "dummy" (key; /v1/data/dummy) value (2)
  - return dummy value right now
  - write config server in go
  - return 404 if value doesnt exist (non-dummy path)
- add PUT endpoint that saves value such that GET endpoint returns it (2)
  - in memory configuration
- allow config server to *fetch* value from a database table (4)
  - add necessary database configuration to the release job (connection url or 2-3 props split out)
  - keep in-memory configuration as a possibility (backend config)
  - assume that a table in the database exists
  - GET endpoint should return a value written to a DB
  - see what concourse does for its db
- allow config server to write value to a database table (1)
  - implement PUT to save into db
  - hopefully we have a little go interface for storing/getting things
- automatically create DB table on config server start (2)
  - value should be text
  - dont create table if already exists
  - see what concourse does

---

- director should parse manifest values looking for {{...}} (2)
  - feature flag it as director.parse_config_values
  - only evaluate properties and env sections
  - return one error per key in the manifest
- director should return errors when config server doesnt have keys (2)
  - hook it up to the config server (https url, ca_cert field, always validate https)
  - return error in the Director task since config server returns 404
- director should replace keys in the manifest with found values from the config server (2)
  - ensure that ERB templates could access replaced info via p and its friends
- dont show config values when diffing manifest (4)
  - even when it's --no-redact
  - show curlies in non redacted case
  - still show lines for changed values
- ensure that director doesnt print interpolated manifest to the debug log (4)
  - download manifest should have curlies as input
  - audit where passwords might be saved into the db
- director should evaluate link information (4)
  - do same as parsing manifest
- director should evaluate runtime config for referenced props during bosh deploy (2)

---

- validate UAA token in the config server
  - only allow "config-server.admin" scope?
- director should include UAA token when making a request to the config server
  - show error from the config server when token is not valid

## TBD

- should links point to specific version of the config value?
- multi value config-server values (eg certificate may contain certificate, private key, and ca certificate)
