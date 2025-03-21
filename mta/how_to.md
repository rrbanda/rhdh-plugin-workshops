# This will guide you configuring a MTA plugin and testing it in RHDH

## Pre-requisites 
* RHDH to be installed already locally or on openshift cluster
* MTA needs to be installed on openshift cluster
* Run the following script to setup client in keycloak of MTA

 ### MTA specific config needed for RHDH
---
` app-config.yaml `

```
auth:
  environment: development
  providers:
    guest:
      dangerouslyAllowOutsideDevelopment: true
    # github:
    #   development:
    #     clientId: ${AUTH_GITHUB_CLIENT_ID}
    #     clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}

# integrations:
#   github:
#     - host: github.com
#       apps:
#         - $include: github-app-credentials.yaml

app:
  title: RHDH
  baseUrl: ${BASE_URL}

mta:
  url: https://mta-openshift-mta.apps.cluster-xvdzv.dynamic.redhatworkshops.io
  backendPluginVersion: "0.3.0"
  backendPluginRoot: "/opt/app-root/src/dynamic-plugins-root/backstage-community-backstage-plugin-mta-backend-0.3.0"
  providerAuth:
    realm: mta
    secret: backstage-provider-secret
    clientID: backstage-provider

dynamicPlugins:
  frontend: 
    ianbolton.backstage-plugin-mta-frontend:
      entityTabs:
        - path: /mta
          title: MTA
          mountPoint: entity.page.mta
      mountPoints:
        - mountPoint: entity.page.mta/cards
          importName: EntityMTAContent
          config:
            layout:
              gridColumn:
                lg: 'span 12'
                md: 'span 8'
                xs: 'span 6'
            if:
              allOf:
                - isKind: component 
                - isType: service
backend:
  listen:
    port: 7007
  baseUrl: ${BASE_URL}
  # uncomment this if backend.baseUrl is exposed over HTTPS
  # and you want RHDH to auto-generate and serve a self-signed certificate.
  # https: true
  cors:
    origin: ${BASE_URL}
    methods: [GET, HEAD, PATCH, POST, PUT, DELETE]
    credentials: true
  csp:
   upgrade-insecure-requests: false

# comment out the following 'database' section to use the PostgreSQL database
  database:
    client: better-sqlite3
    connection: ':memory:'

  auth:
    keys:
      - secret: "development"

# You can use local files from catalog-entities directory to load entities into the catalog
catalog:

  rules:
    - allow: [Component, API, Location, Template, Domain, User, Group, System, Resource]

  locations:
    - type: file
      target: /opt/app-root/src/configs/catalog-entities/users.yaml
      rules:
        - allow: [User, Group]
    - type: file
      target: /opt/app-root/src/configs/catalog-entities/components.yaml
      rules:
        - allow: [Component, System]
    # - type: url
    #   target: https://github.com/backstage/backstage/blob/master/packages/catalog-model/examples/all.yaml
    #   rules:
    #     - allow: [Component, User, Group, Domain]


techdocs:
  generator:
    runIn: local
  builder: local
  publisher:
    type: local
    local:
      publishDirectory: /tmp/techdocs


```

```
#!/bin/bash

# Variables
KEYCLOAK_URL=$(oc get route tackle -n konveyor-tackle -o jsonpath='{.spec.host}')
KEYCLOAK_URL="https://${KEYCLOAK_URL}/auth"
echo "Using Keycloak URL: $KEYCLOAK_URL"

MASTER_REALM="master"
CLIENT_ID="admin-cli"
USERNAME="admin"
MTA_REALM="tackle"

# Fetch the encoded password from the secret
ENCODED_PASSWORD=$(oc get secret tackle-keycloak-sso -n konveyor-tackle -o jsonpath='{.data.admin-password}')

echo "Encoded Password: $ENCODED_PASSWORD"

# Decode the password
PASSWORD=$(echo $ENCODED_PASSWORD | base64 --decode)
echo "Decoded Password: $PASSWORD"


TOKEN=$(curl -s -X POST "$KEYCLOAK_URL/realms/$MASTER_REALM/protocol/openid-connect/token" \
  -d "client_id=$CLIENT_ID" \
  -d "username=$USERNAME" \
  -d "password=$PASSWORD" \
  -d "grant_type=password" | jq -r '.access_token')
echo "Access Token: $TOKEN"

# Define new client JSON
NEW_CLIENT_JSON=$(cat <<EOF
{
  "clientId": "backstage-provider",
  "enabled": true,
  "secret": "backstage-provider-secret",
  "redirectUris": [
    "*"
  ],
  "webOrigins": [],
  "protocol": "openid-connect",
  "attributes": {
    "access.token.lifespan": "900"
  },
  "publicClient": false,
  "bearerOnly": false,
  "consentRequired": false,
  "standardFlowEnabled": true,
  "implicitFlowEnabled": false,
  "directAccessGrantsEnabled": true,
  "serviceAccountsEnabled": true
}
EOF
)

# Create the new client
CREATE_RESPONSE=$(curl -s -X POST "$KEYCLOAK_URL/admin/realms/$MTA_REALM/clients" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$NEW_CLIENT_JSON")
echo "Create Client Response: $CREATE_RESPONSE"

# Get the client ID dynamically
CLIENT_UUID=$(curl -s -X GET "$KEYCLOAK_URL/admin/realms/$MTA_REALM/clients" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" | jq -r '.[] | select(.clientId=="backstage-provider") | .id')
echo "Client UUID: $CLIENT_UUID"

# Fetch client details using the client UUID
CLIENT_DETAILS=$(curl -s -X GET "$KEYCLOAK_URL/admin/realms/$MTA_REALM/clients/$CLIENT_UUID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json")
echo "Client Details: $CLIENT_DETAILS"

# Fetch service account user ID for the client
SERVICE_ACCOUNT_USER_ID=$(curl -s -X GET "$KEYCLOAK_URL/admin/realms/$MTA_REALM/clients/$CLIENT_UUID/service-account-user" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" | jq -r '.id')
echo "Service Account User ID: $SERVICE_ACCOUNT_USER_ID"

# Fetch role IDs
TACKLE_ADMIN_ROLE_ID=$(curl -s -X GET "$KEYCLOAK_URL/admin/realms/$MTA_REALM/roles/tackle-admin" \
  -H "Authorization: Bearer $TOKEN" | jq -r '.id')
echo "Tackle Admin Role ID: $TACKLE_ADMIN_ROLE_ID"

DEFAULT_ROLES_MTA_ID=$(curl -s -X GET "$KEYCLOAK_URL/admin/realms/$MTA_REALM/roles/default-roles-tackle" \
  -H "Authorization: Bearer $TOKEN" | jq -r '.id')
echo "Default Roles MTA ID: $DEFAULT_ROLES_MTA_ID"

# Prepare role assignment JSON using the fetched IDs
ASSIGN_ROLES_PAYLOAD=$(cat <<EOF
[
  {
    "id": "$TACKLE_ADMIN_ROLE_ID",
    "name": "tackle-admin"
  },
  {
    "id": "$DEFAULT_ROLES_MTA_ID",
    "name": "default-roles-tackle"
  }
]
EOF
)
echo "Assign Roles Payload: $ASSIGN_ROLES_PAYLOAD"


# Assign roles to the service account
ASSIGN_ROLES_RESPONSE=$(curl -s -X POST "$KEYCLOAK_URL/admin/realms/$MTA_REALM/users/$SERVICE_ACCOUNT_USER_ID/role-mappings/realm" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$ASSIGN_ROLES_PAYLOAD")
echo "Assign Roles Response: $ASSIGN_ROLES_RESPONSE"

echo "Roles assigned to the service account of client 'backstage-provider'."

DESIRABLE_SCOPES=(
  "acr" "addons:delete" "addons:get" "addons:post" "addons:put" "address"
  "adoptionplans:post" "analyses:delete" "analyses:get" "analyses:post" "analyses:put"
  "applications.analyses:delete" "applications.analyses:get" "applications.analyses:post" "applications.analyses:put"
  "applications.assessments:get" "applications.assessments:post" "applications.bucket:delete" "applications.bucket:get"
  "applications.bucket:post" "applications.bucket:put" "applications.facts:delete" "applications.facts:get"
  "applications.facts:post" "applications.facts:put" "applications.stakeholders:put" "applications.tags:delete"
  "applications.tags:get" "applications.tags:post" "applications.tags:put" "applications:delete" "applications:get"
  "applications:post" "applications:put" "archetypes.assessments:get" "archetypes.assessments:post"
  "archetypes:delete" "archetypes:get" "archetypes:post" "archetypes:put" "assessments:delete" "assessments:get"
  "assessments:post" "assessments:put" "buckets:delete" "buckets:get" "buckets:post" "buckets:put"
  "businessservices:delete" "businessservices:get" "businessservices:post" "businessservices:put" "cache:delete"
  "cache:get" "dependencies:delete" "dependencies:get" "dependencies:post" "dependencies:put" "email"
  "files:delete" "files:get" "files:post" "files:put" "identities:delete" "identities:get" "identities:post"
  "identities:put" "imports:delete" "imports:get" "imports:post" "imports:put" "jobfunctions:delete"
  "jobfunctions:get" "jobfunctions:post" "jobfunctions:put" "microprofile-jwt" "migrationwaves:delete"
  "migrationwaves:get" "migrationwaves:post" "migrationwaves:put" "offline_access" "phone" "profile"
  "proxies:delete" "proxies:get" "proxies:post" "proxies:put" "questionnaires:delete" "questionnaires:get"
  "questionnaires:post" "questionnaires:put" "reviews:delete" "reviews:get" "reviews:post" "reviews:put"
  "roles" "rulesets:delete" "rulesets:get" "rulesets:post" "rulesets:put" "settings:delete" "settings:get"
  "settings:post" "settings:put" "stakeholdergroups:delete" "stakeholdergroups:get" "stakeholdergroups:post"
  "stakeholdergroups:put" "stakeholders:delete" "stakeholders:get" "stakeholders:post" "stakeholders:put"
  "tagcategories:delete" "tagcategories:get" "tagcategories:post" "tagcategories:put" "tags:delete" "tags:get"
  "tags:post" "tags:put" "targets:delete" "targets:get" "targets:post" "targets:put" "tasks.bucket:delete"
  "tasks.bucket:get" "tasks.bucket:post" "tasks.bucket:put" "tasks:delete" "tasks:get" "tasks:post"
  "tasks:put" "tickets:delete" "tickets:get" "tickets:post" "tickets:put" "trackers:delete" "trackers:get"
  "trackers:post" "trackers:put"
)


# Fetch all client scopes
CLIENT_SCOPES=$(curl -s "$KEYCLOAK_URL/admin/realms/$MTA_REALM/client-scopes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json")

# Define desired scopes

# Find the IDs of the desired scopes and store them in a string
SCOPE_IDS=""
for SCOPE_NAME in "${DESIRABLE_SCOPES[@]}"; do
  SCOPE_ID=$(echo $CLIENT_SCOPES | jq -r --arg SCOPE_NAME "$SCOPE_NAME" '.[] | select(.name == $SCOPE_NAME) | .id')
  if [ -n "$SCOPE_ID" ]; then
    SCOPE_IDS="$SCOPE_IDS $SCOPE_ID"
    echo "$SCOPE_NAME ID: $SCOPE_ID"
  else
    echo "Error: Scope $SCOPE_NAME not found."
  fi
done

# Client UUID
CLIENT_UUID=$(curl -s -X GET "$KEYCLOAK_URL/admin/realms/$MTA_REALM/clients?clientId=backstage-provider" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" | jq -r '.[0].id')
echo "Client UUID: $CLIENT_UUID"

# Assign the found scopes to the client
for SCOPE_ID in $SCOPE_IDS; do
  ASSIGN_SCOPE_RESPONSE=$(curl -s -X PUT "$KEYCLOAK_URL/admin/realms/$MTA_REALM/clients/$CLIENT_UUID/default-client-scopes/$SCOPE_ID" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json")
  echo "Assigned Scope ID $SCOPE_ID to client $CLIENT_UUID"
done

echo "Assigned desired scopes to the client 'backstage-provider'."

```
---

` dynamic-plugins.yaml `

```
includes:
  - dynamic-plugins.default.yaml
plugins: 
      - disabled: false
        package: "@backstage-community/backstage-plugin-mta-backend@0.3.0"
        integrity: sha512-gEmEEa1Fuh+jRqGiLg5JgrBGj+rdRxohnGuIcXb6iTuv2kG0gqkLHu/U1smm+rdBZ78h8lnbrCsEMPKwvjWs+A== 
      - disabled: false
        package: "@ianbolton/backstage-plugin-mta-frontend@0.1.8"
        integrity: sha512-OIzO+6HMAv78sN4a6LjB928AX0StJXY3PlhkXJRt/Er+AO+2m2MGvTkS0QzZGvoBoquwQjV+s+CFWDnhrGCAAQ==
      - disabled: false
        package: "@backstage-community/backstage-plugin-catalog-backend-module-mta-entity-provider@0.2.0"
        integrity: sha512-M2s8SN3/tgav+1q1RWVgxAA0V/bRFeT4GesMMWh7/WpFcrqAe65fUJ1YrTPeDavtjW27YE/RZCjwdM9xtPoc5w== 
      - disabled: false
        package: "@backstage-community/backstage-plugin-scaffolder-backend-module-mta@0.3.0"
        integrity: sha512-dGdjnBGmmMtvQ46LF1QUzlu+C1fnLyI0iljTJfwToOiZOxfb1vl/1gsPfT74kLSit1fYLPm8N+weS0i5+NqS8A==

```

### Note

I have shared RHDH app config completely above but look for the MTA specifics i.e from Line 33 to 61 above


## Run MTA from Red Hat Developer Hub UI



### Access Developer Hub 

<img width="1508" alt="Screenshot 2025-03-20 at 10 18 37 PM" src="https://github.com/user-attachments/assets/f79d7c2a-9e94-4e40-ab13-4469646d89c0" />


### Register Template 

<img width="1508" alt="Screenshot 2025-03-20 at 10 20 58 PM" src="https://github.com/user-attachments/assets/60d552a6-c7bb-4f41-818f-c21b3c606ed6" />

### Verify in RHDH Catalog for the Catalog entry 

<img width="1508" alt="Screenshot 2025-03-20 at 10 22 37 PM" src="https://github.com/user-attachments/assets/ed6bcb3d-9add-402f-bb80-efcaeb5d7dd3" />



### Create Application 

<img width="1508" alt="Screenshot 2025-03-20 at 10 22 37 PM" src="https://github.com/user-attachments/assets/1319ef91-a256-40a4-88b5-2cd0118722dd" />
<img width="1508" alt="Screenshot 2025-03-20 at 10 23 26 PM" src="https://github.com/user-attachments/assets/c05e6e96-01a3-46e7-9fd5-a785af3f1554" />
<img width="1508" alt="Screenshot 2025-03-20 at 10 23 37 PM" src="https://github.com/user-attachments/assets/917a00eb-a542-4b89-911e-b82b3385fcb9" />
<img width="1508" alt="Screenshot 2025-03-20 at 10 23 45 PM" src="https://github.com/user-attachments/assets/6459e10d-9d63-4ae5-8e0d-a94bd7a9cbab" />


### Perform Analysis

<img width="2557" alt="Screenshot 2025-03-20 at 8 50 13 PM" src="https://github.com/user-attachments/assets/f4e87581-1eb9-4f7c-929b-d23b94458aa7" />

### View MTA Analysis status in RHDH

<img width="2557" alt="Screenshot 2025-03-20 at 8 50 13 PM" src="https://github.com/user-attachments/assets/3586774f-15f9-43e7-b683-4fe13a719fb2" />


### View Analysis status in MTA 

<img width="2598" alt="Screenshot 2025-03-20 at 8 51 56 PM" src="https://github.com/user-attachments/assets/5aba2bca-10b6-43c8-b880-5bcf5a3ea9ee" />



### Verify status in RHDH
<img width="1512" alt="Screenshot 2025-03-20 at 9 47 33 PM" src="https://github.com/user-attachments/assets/195cc34f-6b54-49a2-9332-033ac371a936" />

