# A basic deployment of Ontoserver4, Atomio and Ontocloak

This project intends to give a basic rundown (not production ready) deployment that can be used to learn how Ontoserver, Atomio and Ontocloak can be configured together, to give you a starting point for your own deployment/experimentation. It is built around images that are hosted at quay.io/repository/aehrc. To get access to these images you will need to:
- Establish an account with quay.io at https://quay.io
- Obtain an appropriate Licence:
- Within Australia, email help@digitalhealth.gov.au to request a (free) Ontoserver licence. ADHA will then arrange authorisation for your quay.io account.
- Elsewhere, email ontoserver-support@csiro.au to discuss licensing terms (both evaluation and production licences are available for single and multiple instances, no limit on number of users). Once the licence is established, CSIRO will register your quay.io account name to enable access to their repository

:warning: To run this on an M1/2 chip you will need to enable rosetta support on docker - to do this as of 12/07/23 go to docker desktop -> settings -> features in development -> 'Use Rosetta for x86/amd64 emulation on Apple Silicon'

:warning: You should go through the set up from start to finish - as there is a few steps that are necassary to make the stack run, it will not run with a simple docker compose up

### Ontocloak

Ontocloak is a wrapper around keycloak, that intends to make it easier to use keycloak with ontoserver.

You can read more about Ontocloak [here](https://ontoserver.csiro.au/site/our-solutions/ontocloak/) and view the technical documentation [here](https://ontoserver.csiro.au/site/technical-documentation/ontocloak-documentation/) and [here](https://ontoserver.csiro.au/docs/6/config-all.html).

A realm file is provided at realms/onto-realm.json that provides some sensibles defaults for a basic ontocloak setup that can be used to secure ontoserver and atomio. A good tutorial that was written from the perspective of someone using keycloak for the first time can be found [here](https://github.com/itcr-uni-luebeck/ontoserver-keycloak) although it is a little out of date, it is a great place to start.


### Ontoserver

You can read more about Ontoserver [here](https://ontoserver.csiro.au/site/our-solutions/ontoserver/) and view the technical documentation [here](https://ontoserver.csiro.au/docs/6/).

### Atomio

You can read more about Atomio [here](https://ontoserver.csiro.au/site/our-solutions/ontoserver/) and view the technical documentation [here](https://ontoserver.csiro.au/docs/6/).

### Putting it all together

Most of the setup is handled within the docker-compose file, with some steps needing to be done manually for the first time. Namely creating the self signed certificates which is mentioned [here](#ontocloak--its-postgres-setup) and [here](#ontoserver-setup) and generating some keycloak user's that is mentioned [here](#postman)

##### /etc/hosts setup

You will need to map the servers to localhost in /etc/hosts (or wherever the hosts file is located on your machine, on a windows machine this is usually located at C:\Windows\System32\drivers\etc\hosts) To do this on mac and linux $sudo vi /etc/hosts and add the line

```
127.0.0.1   localhost ontocloakdemo ontoserver atomio
```

After the completion of setup the servers will be available at:
- https://ontocloakdemo:8443
- https://ontoserver:8444
- http://atomio:3000
  
##### Ontocloak & it's postgres setup

To run Ontocloak, you will need to create some self signed certificates, you can do this using openssl from the root of the project. The reason that ontocloak is ran over https is that atomio will not accept a security provider over http.
As these are self signed certificates, atomio/ontoserver won't trust these by default so we need to create a trust store, which will then be mounted into the atomio/ontoserver containers.

:warning: Enter 'ontocloakdemo' when openssl asks for the domain name

```
openssl genrsa -out ./certs/ontocloak/server.key 2048
openssl req -new -out ./certs/ontocloak/server.csr -key ./certs/ontocloak/server.key
openssl x509 -req -days 365 -in ./certs/ontocloak/server.csr -signkey ./certs/ontocloak/server.key -out ./certs/ontocloak/server.crt
## Create a truststore that will be mounted into the atomio container
keytool -import -alias ontocloak_crt -file ./certs/ontocloak/server.crt -noprompt -storepass changeit -keystore ./certs/ontocloak/truststore
```


```
ontocloakdemo:
    platform: linux/amd64
    image: quay.io/aehrc/ontocloak:4.1
    command:
      - start
      - --import-realm
      # --optimized is needed for keycloak to function properly
      - --optimized
    environment:
      KC_DB: ${KC_DB}
      KC_DB_URL_HOST: postgres_ontocloak_demo
      KC_DB_URL_DATABASE: ${KC_DB_URL_DATABASE}
      KC_DB_PASSWORD: ${KC_DB_PASSWORD}
      KC_DB_USERNAME: ${KC_DB_USERNAME}
      KC_SCHEMA: ${KC_SCHEMA}
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_HOSTNAME_STRICT: 'false'
      KC_HTTPS_CERTIFICATE_FILE: /etc/x509/https/tls.crt
      KC_HTTPS_CERTIFICATE_KEY_FILE: /etc/x509/https/tls.key
      KEYCLOAK_FRONTEND_URL: https://localhost:8443/auth
      JAVA_OPTS_APPEND: -Dontocloak.action.agreement.enduser.version=1.0 -Dontocloak.action.agreement.enduser.title='Acme End User Agreement v1.0' -Dontocloak.action.agreement.enduser.html_text='Some end user agreement text' -Dontocloak.action.agreement.customer.version=1.0 -Dontocloak.action.agreement.customer.title='Acme Customer Agreement v1.0' -Dontocloak.action.agreement.customer.html_text='Some customer agreement text'
    ports: 
      - "8443:8443"
    volumes: 
        - ./certs/ontocloak/server.crt:/etc/x509/https/tls.crt
        - ./certs/ontocloak/server.key:/etc/x509/https/tls.key
        - ./realms/onto-realm.json:/opt/keycloak/data/import/onto-realm.json:ro

    depends_on:
      postgres_ontocloak_demo:
        condition: service_healthy
    networks:
      - ontocloak_demo_dev_network
  
  postgres_ontocloak_demo:
    image: postgres:14.2
    command: postgres -c 'max_connections=200'
    volumes: 
      - pgdata_ontocloak_demo:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${KC_DB_URL_DATABASE}
      POSTGRES_USER: ${KC_DB_USERNAME}
      POSTGRES_PASSWORD: ${KC_DB_PASSWORD}
    healthcheck:
      test: "exit 0"
    ports:
      - "5436:5432"
    networks:
      - ontocloak_demo_dev_network
```

Some important things to note in this part of the docker-compose is the KC_PASSWORD,_USERNAME,_DB and POSTGRES_PASSWORD,_USERNAME,_DB match.
KEYCLOAK_ADMIN and KEYCLOAK_ADMIN_PASSWORD are the passwords for the administrator account that will be created automatically on first start for ontocloak. These are configurable in the .env file

All user's created with the example realm file by default have ontoserver read rights. If you want to add write rights to a user - you will need to add the role system/*.write to the user. Similarly, if you want a user with atomio rights, you will need to add SYND_READ/WRITE roles to the user.

Make sure you are in the ontoserver realm and not the master realm.

##### Atomio setup

The only thing really of note here, that is outside the normal setup is the JAVA_TOOL_OPTIONS. This needs to be passed to allow atomio to trust self signed certificates. The Dockerfile located at /atomio/Dockerfile takes the certificate that is located at /certs/ontocloak/server.crt and creates a truststore in the atomio container.

Atomio will be discoverable at http://atomio:3000

```
atomio:
    platform: linux/amd64
    image: quay.io/aehrc/atomio:1
    container_name: atomio
    environment:
      # The 'aud' in the token generated by keycloak for this client
      - atomio.security.audience=atomio
      - atomio.security.hsts=true
      # Whether this server is to be secured by tokens
      - atomio.security.enabled=true
      - atomio.security.anonymousFeedRead=true
      - atomio.cors.allowedOriginPatterns=ALL
      # The location of the issuer of tokens
      - atomio.security.issuer-uri=${ATOMIO_REALM_ENDPOINT}
      - atomio.client.urlWhitelist=http://atomio:8080, https://ontoserver:443
      # Passing java opts to atomio - useful in our case to tell the jvm to trust ontocloaks self signed certificate
      - JAVA_TOOL_OPTIONS=-Djavax.net.ssl.trustStore=/usr/lib/jvm/java-17-openjdk/lib/security/truststore -Djavax.net.ssl.trustStorePassword=changeit
      # Configure atomio to talk to itself - needed to clone one of it's own feeds!
      - atomio.security.client.1.url_prefix=${ATOMIO_BASE_URL}
      - atomio.security.client.1.client_id=${ATOMIO_CLIENT_ID}
      - atomio.security.client.1.client_secret=${ATOMIO_CLIENT_SECRET}
      - atomio.security.client.1.token_prefix=${ONTOCLOAK_ONTOSERVER_TOKEN_ENDPOINT}
      # Configure atomio to talk to ontoserver - needed to clone a feed from ontoserver!
      - atomio.security.client.2.url_prefix=${ONTOSERVER_BASE_URL}
      - atomio.security.client.2.client_id=${ONTOSERVER_CLIENT_ID}
      - atomio.security.client.2.client_secret=${ONTOSERVER_CLIENT_SECRET}
      - atomio.security.client.2.token_prefix=${ONTOCLOAK_ONTOSERVER_TOKEN_ENDPOINT}
    depends_on:
      - ontocloakdemo
    ports:
      - "3000:8080"
    volumes: 
        - ./certs/ontocloak/truststore:/usr/lib/jvm/java-17-openjdk/lib/security/truststore
    networks:
      - ontocloak_demo_dev_network
    restart: always
```

##### Ontoserver setup 

Ontoserver can't (currently) discover it's security settings in the same manner as Atomio, so instead you need to copy the rs256 public key from the ontoserver realm in keycloak.
Similarly you need to get the client secret for the atomio client within the ontoserver realm of keycloak.

Alot of the environment values are commented, so those values will not be expanded upon here. You can find a full lost of config values for ontoserver [here](https://ontoserver.csiro.au/docs/6/config-all.html).

You will need to provide some certificates to ontocache inside ontocache/certs. You will need privkey.pem and fullchain.pem
These can be generated via openssl

```
openssl genrsa > ./ontocache/certs/privkey.pem
openssl req -new -x509 -key ./ontocache/certs/privkey.pem > ./ontocache/certs/fullchain.pem
```

```
ontoserver:
    image: quay.io/aehrc/ontoserver:ctsa-6
    container_name: ontoserver-stack
    depends_on:
      - ontocloakdemo
      - postgres_ontocloak_demo
      - db
    ports:
      - "5005:5005"
    environment:
      # A list of all environment configs is available here: https://ontoserver.csiro.au/docs/6/config-all.html
      - JAVA_OPTS=-Xmx8G -Djavax.net.ssl.trustStore=/usr/lib/jvm/java-17-openjdk/lib/security/truststore -Djavax.net.ssl.trustStorePassword=changeit -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
      - ontoserver.fhir.base=https://ontoserver:8444/fhir
      - ontoserver.fhir.header.cacheControl=must-revalidate,max-age=1
      # enables role based security in ontoserver
      - ONTOSERVER_INSECURE=true
      # - ontoserver.security.enabled=true
      # readonly settings for each endpoint, by default this is set to false
      - ontoserver.security.readOnly.api=true
      - ontoserver.security.readOnly.fhir=true
      - ontoserver.security.readOnly.synd=true
      - ontoserver.security.token.secret=${ONTOSERVER_SECURITY_SECRET}
      - conformance.security.description=This is a description that you can use to describe how oauth works with this server
      # this is featured in the machine-readable ConformanceStatement and should point to the "authorize" url of your keycloak installation
      - conformance.security.authorize=https://ontocloakdemo:8443/auth/realms/ontoserver/protocol/openid-connect/auth 
      # The token endpoint
      - conformance.security.token=${ONTOCLOAK_ONTOSERVER_TOKEN_ENDPOINT}
      - conformance.security.kinds=SMART-on-fhir
      - conformance.security.audience=https://ontoserver:8444/fhir 
      - ontoserver.security.verify.certs=false
      # The location of the upstream syndication server
      - atom.syndication.feedLocation=${ATOMIO_FEED_LOCATION}
      # Hostname (including protocol, e.g. http:// or https://) for which to provide OAuth2 credentials when requesting content
      - authentication.oauth.endpoint.1=${ATOMIO_BASE_URL}
      # OAuth client ID to send when requesting a system token for use in requesting content from authentication.oauth.endpoint.{n}
      - authentication.oauth.endpoint.client_id.1=${ATOMIO_CLIENT_ID}
      # Client secret to send when requesting an OAuth2 system token for use in requesting content from authentication.oauth.endpoint.{n}
      - authentication.oauth.endpoint.client_secret.1=${ATOMIO_CLIENT_SECRET}
      #  Strategy (basic_auth or body) to use when requesting an OAuth2 system token for use in requesting content authentication.oauth.endpoint.{n}
      - authentication.oauth.endpoint.strategy.1=body
      # Token endpoint URL for requesting an OAuth token for use in retrieving content from authentication.oauth.endpoint.{n}
      - authentication.oauth.endpoint.token_endpoint.1=${ONTOCLOAK_ONTOSERVER_TOKEN_ENDPOINT}
      - TZ=${TZ}
    volumes:
      - onto-data:/var/onto
      - ./certs/ontocloak/truststore:/usr/lib/jvm/java-17-openjdk/lib/security/truststore
    logging:
      options:
        max-size: 1024m
    networks:
      - ontocloak_demo_dev_network
```

##### .env Setup

:warning: You will need to configure the values in the .env file.

You should start by running docker compose up, as you will need to access ontocloakdemo to be able to get some of the values for the .env file.

The values that you will need to update here are ONTOSERVER_SECURITY_SECRET, ONTOSERVER_CLIENT_SECRET, ATOMIO_CLIENT_SECRET.

ONTOSERVER_SECURITY_SECRET can be found on keycloak by following realm settings -> keys -> rs 256 -> public key.

ONTOSERVER_CLIENT_SECRET & ATOMIO_CLIENT_SECRET secrets can be found for the client's on ontocloakdemo by following clients -> {clientName} -> credentials -> secret.
These secrets will also need to be put into the postman setup.

You can get these secrets at these urls [ontoserver](https://ontocloakdemo:8443/auth/admin/master/console/#/realms/ontoserver/clients/86caea36-9836-44ae-9c95-25b8003bc554/credentials),
[atomio](https://ontocloakdemo:8443/auth/admin/master/console/#/realms/ontoserver/clients/b51e3771-c86e-4ad1-9c30-daa9b3585882/credentials)

```
# Set up for ontocloaks postgres
TZ=Australia/Brisbane
KC_DB=postgres
KC_DB_URL_DATABASE=keycloak
KC_DB_PASSWORD=password
KC_DB_USERNAME=keycloak
KC_SCHEMA=public
# The admin login for ontocloak, enter these credentials in the management console of ontocloak
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=admin

# Passed to atomio, the end point where atomio can self configure oauth
ATOMIO_REALM_ENDPOINT=https://ontocloakdemo:8443/auth/realms/ontoserver

# Passed to ontoserver, the secret of the 'ontoserver' realm on ontocloak
# This needs to be the public key for your realm, go to https://ontocloakdemo:8443/auth/admin/master/console/#/ontoserver/realm-settings/keys and copy the public key from rs256
# It must include the BEGIN and END
ONTOSERVER_SECURITY_SECRET=-----BEGIN PUBLIC KEY-----MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAnYW8O6vM1Yb0hL41AVIAq63IcEmF57LzFaWKZabLLrqjcZxjQTwnCLKPe2+PE7Gynb10F35efIv+SSIWLc6G9O1Fex3zqchXLAq+V7l5EIaK8rXl9Y0qrDonddAhHdDe8TS4i/DUki3VoGfhbwa3HnZGmAre1VnsYfEIrLuv9A5eOYI56cq13U6ev9rrLhVvX7yQ8WjzMi1z1W8C4OcrCxZLMIWxiDBoFtefVk4qY4uYkcTDEmzKmNkb0cFF/2f1JrsWObC3mM5pJcgN7t+57Ag9EyMEwBYk8VQT5u7vJuYx99KceZWtNYzcfrvYPR8iUZMj0YULOXUItNFlBDSYzQIDAQAB-----END PUBLIC KEY-----

ONTOSERVER_BASE_URL=https://ontoserver:8444
ONTOSERVER_CLIENT_ID=ontoserver
# You may need to change this secret
ONTOSERVER_CLIENT_SECRET=TRhuWVeCasPbWYjfzeJHgjjmfYPapMf8

# The url of the upstream syndication feed - in this case the atomio we have created, and the location of the upstream feed
ATOMIO_BASE_URL=http://atomio:8080
ATOMIO_FEED_LOCATION=http://atomio:8080/feed/testFeed/syndication.xml

# Passed to ontoserver & atomio- the values from ontocloak for the 'atomio' client
# This allows ontoserver & atomio to read/write to atomio
ATOMIO_CLIENT_ID=atomio
# You may need to change this secret
ATOMIO_CLIENT_SECRET=AHinacPgb3MPVSq6bn8EjzzjUOyKNxrL
ONTOCLOAK_ONTOSERVER_TOKEN_ENDPOINT=https://ontocloakdemo:8443/auth/realms/ontoserver/protocol/openid-connect/token
```

After updating these variables in the .env, run docker compose down && docker compose up

### Postman and checking out the environment

In the postman folder you will find some environments, and example requests to do the things that you will need client tokens obtained from keycloak. There is two environments - one for ontoserver, and one for atomio.

![Environment Variables](/images/environment_variables.png "Environment Variables")  

You may need to replace some of these environment variables with the ones from your setup, mainly the client_secret for each environment. These can be obtained on keycloak under clients -> {client name} -> credentials you can regenerate the client secret to obtain it.
This is the same as the ONTOSERVER_CLIENT_SECRET and ATOMIO_CLIENT_SECRET that you have setup in your .env file.

![Environment Folder Structure](/images/folder_structure.png "Folder Structure")  

Each of the folder's themselves have tokens set on them, and to request a token for said folder change your environment to either ontoserver/atomio and make the request to keycloak.

:warning: unnecassary step, only do the next paragraph if desired

You may wish to login with a user that you have created, and granted whatever read/write privelages you want. To do this, you will need to create a user on ontocloak, and grant it the desired roles that you want it to have. Note the change of Grant type to Authorization Code (With PKCE), and Authorize Using Browser being ticked.
This is an example of the structure of a request using the browser authentication flow, where you are logging in with a specific user.

![Example token request](/images/tokens.png "Token Request")  

A summary of the things that will be done, which are detailed in the numbered steps.

- Create a new feed in atomio through {{atomio}}/feed
- Create an entry in the feed you just created through {{atomio}}/feed/{{feedName}}
- Pull that feed into ontoserver through {{ontoserver}}/api/upstream.xml
- Go to ontocommand, import that file and view it in Shrimp/snapper

1. First, ensure that your containers are running through a docker compose up in the root dir of this project.  

2. Next, head to postman and open up our environments and collection. Set the values in the postman environments with your client secrets.   
   
3. Get a token for the atomio collection. 

:warning: Note, you will need to be using the atomio environment for this step until step 8.

4. Send the get feeds request - note that we have no feeds.    

![Empty feeds](/images/empty_feeds.png)    

5. Send the create feeds request, and get feeds again. We now have a feed named testFeed   
   
![Single Feed](/images/single_entry_feed.png)  

6. Send the add feed entry request. To do this you will link to the file codesystem-ucom.entry.json to the entry field in the request, and codesystem-ucom.json to the file field in the request. Both of these files are located in the postman directory  

7. Send the get individual feed request - to see the codesystem we just added to atomio   

![Feed with entry](/images/single_entry_feed.png)  

We will now pull this feed into ontoserver  

8. Select the ontoserver environment, and request a token for ontoserver through the ontoserver collection

9. Make the get upstream feed request in the ontoserver collection

![Feed pulled into ontoserver](/images/upstream_feed.png)

### We will now checkout the ontoserver through ontocommand environment. 

1. head to https://ontoserver:8444/fhir
   
2. Go to syndication -> import the UCUM entry that we have added to ontoserver

![Ontoserver syndication](/images/ontoserver_ucum.png)
   


