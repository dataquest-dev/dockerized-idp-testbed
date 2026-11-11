# dockerized-idp-testbed

Provides a shibboleth-sp, shibboleth-idp setup that should help testing an app that needs to provide shibboleth auth. The idea is to tweak the sp & httpd proxy configs to resemble your expected shibbolethized setup. The preexisting users might help covering some edge cases in your app.

Based on https://github.com/UniconLabs/dockerized-idp-testbed; the setup for the shibboleth-idp image used is at https://github.com/Unicon/shibboleth-idp-dockerized

Start at https://localhost/ and test from there.


## What's there
A docker-compose setup that runs
 - shibboleth-sp
 - 2 shibboleth-idp instances
 - httpd as a proxy
 - ldap
 - mysql

### shibboleth-idp
- uses the ldap as a source of "users"
- the idps mostly have the same config except for entity ids and some minor details.
- Attribute release: idp `https://someother/idp/shibboleth` does not release eppn (`https://idptestbed/idp/shibboleth` does)

### shibboleth-sp
- configured to run with apache httpd
- the container also provides basic idp discovery, which allows users to pick from the two idps

### mysql
- only needed to store persistent-id (eduPersonTargetedID)

### ldap
- see `ldap/users.ldif` for the list of users and their passwords (always `password`). Use the `uid` as login

### httpd-proxy
- provides a simple index page that links to a protected resource `/php-shib-protected/`; that's just `phpinfo()` which nicely shows all the attributes released/mapped by the idp/sp.

## Not using localhost
The idp and sp know each other's metadata. These contain the url bindings where they expect communication. These urls must be reacheable from user's browser. Also the sp makes some assumptions based on the servername (configured in httpd).
You can change the `.env` file (there's just `HOSTNAME_FOR_BROWSER`) and `docker-compose build`. This should be the hostname of the httpd-proxy container. The name must be resolvable at the test machine - where the browser runs. The project originally used `idptestbed` as the hostname; that's the default if there's no `HOSTNAME_FOR_BROWSERS`. Originally, it was suggested to modify `/etc/hosts`.


> **Security:** the IdP signing/encryption keys in `secrets/` are public test keys (upstream UniconLabs testbed). Anyone can forge a login from `https://idptestbed/idp/shibboleth` or `https://someother/idp/shibboleth`. Only trust these IdPs on test SPs with no real data and no real admin accounts.

## Registered SPs
Both IdPs (`idp`, `idp2`) trust these SPs (`shibboleth-idp/conf/metadata-providers.xml`):

| SP entityID | metadata file | what |
|---|---|---|
| `https://dev-5.pc/shibboleth-sp` | `metadata/sp-metadata.xml` | dev-5.pc:8443 nginx+shibboleth |
| `https://dev-20.pc:8443/shibboleth-sp` | `metadata/sp-dev20-metadata.xml` | dev-20.pc:8443 SP gateway for all dev-20 repositories, see https://github.com/dataquest-dev/DSpace/wiki/Shibboleth |

Add an SP:
1. Save its `https://<sp-host>/Shibboleth.sso/Metadata` as `metadata/sp-<name>-metadata.xml` in **both** `idp/shibboleth-idp/` and `idp2/shibboleth-idp/`.
2. Add one line next to `TestSP` in **both** `shibboleth-idp/conf/metadata-providers.xml`:
   `<MetadataProvider id="SP-<name>" xsi:type="FilesystemMetadataProvider" metadataFile="%{idp.home}/metadata/sp-<name>-metadata.xml"/>`
3. Attributes: `attribute-filter.xml` policies use `ANY`, so a new SP gets the same attributes as the others (idp2 never releases eppn).
4. Rebuild and recreate one IdP at a time (`restart` is not enough, files are baked into the image):
   ```sh
   docker compose build idp idp2
   docker compose up -d --no-deps idp
   # test an SP-initiated login through idp, then:
   docker compose up -d --no-deps idp2
   # test an SP-initiated login through idp2
   ```
