
# The following instructions will help configure and test MTA Plugin in Red Hat Developer Hub.

## Prerequisites

Before you begin:

- Ensure **RHDH** is installed locally or on an OpenShift cluster.
- **Migration Toolkit for Applications (MTA)** must be installed on the OpenShift cluster.
- Run the provided script to configure the MTA client in Keycloak.

---

## Installing MTA on OpenShift

Follow the official documentation for **MTA 7.2**:

📚 [Migration Toolkit for Applications 7.2 Documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_applications/7.2/)

🔧 [User Interface Guide (includes operator installation steps)](https://docs.redhat.com/en/documentation/migration_toolkit_for_applications/7.2/html/user_interface_guide/index)

---

## Configuration: RHDH and MTA Plugin


The following section is specific to the MTA plugin configuration that you need configure in `app-config.yaml`

```yaml
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
```

---

## MTA Plugin Setup Script for Keycloak

Use the following script to configure the `backstage-provider` client and assign necessary roles/scopes.



```
#!/bin/bash

# Variables
KEYCLOAK_URL=$(oc get route mta -n openshift-mta -o jsonpath='{.spec.host}')
KEYCLOAK_URL="https://${KEYCLOAK_URL}/auth"
echo "Using Keycloak URL: $KEYCLOAK_URL"

MASTER_REALM="master"
CLIENT_ID="admin-cli"
USERNAME="admin"
MTA_REALM="mta"

# Fetch the encoded password from the secret
ENCODED_PASSWORD=$(oc get secret credential-mta-rhsso -n openshift-mta -o jsonpath='{.data.ADMIN_PASSWORD}')
echo "Encoded Password: $ENCODED_PASSWORD"

# Decode the password
PASSWORD=$(echo $ENCODED_PASSWORD | base64 --decode)
echo "Decoded Password: $PASSWORD"

# Obtain access token using the decoded password
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

DEFAULT_ROLES_MTA_ID=$(curl -s -X GET "$KEYCLOAK_URL/admin/realms/$MTA_REALM/roles/default-roles-mta" \
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
    "name": "default-roles-mta"
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


# Fetch all client scopes
CLIENT_SCOPES=$(curl -s "$KEYCLOAK_URL/admin/realms/$MTA_REALM/client-scopes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json")

# Define desired scopes
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

### `dynamic-plugins.yaml` Configuration

#### Ensure your plugin list includes the following entries:

```yaml
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

---

## Using MTA in Red Hat Developer Hub

### 1. Access Developer Hub

Navigate to your running instance of **Red Hat Developer Hub** in the browser.

![Access Developer Hub](https://github.com/user-attachments/assets/f79d7c2a-9e94-4e40-ab13-4469646d89c0)

---

### 2. Register a Template

On the RHDH UI , Click register existing component to import MTA template available at https://github.com/backstage/community-plugins/blob/main/workspaces/mta/create-application-template.yaml 

![Register Template](https://github.com/user-attachments/assets/60d552a6-c7bb-4f41-818f-c21b3c606ed6)

---

### 3. Verify Template in Catalog

Click **Catalog** and look for the registered template appears as an entity as shown below

![Catalog Entry](https://github.com/user-attachments/assets/ed6bcb3d-9add-402f-bb80-efcaeb5d7dd3)

---

### 4. Create an MTA Application

Choose the template Onboard Application to MTA and perform as shown in the following screenshots.

![Step 1](https://github.com/user-attachments/assets/1319ef91-a256-40a4-88b5-2cd0118722dd)

* Add the name of the application to be onboarded to MTA
  
![Step 2](https://github.com/user-attachments/assets/c05e6e96-01a3-46e7-9fd5-a785af3f1554)

* Fill Application Metadata
  
![Step 3](https://github.com/user-attachments/assets/917a00eb-a542-4b89-911e-b82b3385fcb9)

* Confirm and Submit
  
![Step 4](https://github.com/user-attachments/assets/6459e10d-9d63-4ae5-8e0d-a94bd7a9cbab)

---

### 5. Run MTA Analysis

Once the application is created, go to the entity page and launch an MTA analysis.

![MTA Analysis Start](https://github.com/user-attachments/assets/f4e87581-1eb9-4f7c-929b-d23b94458aa7)

---

### 6. Check Analysis Status in RHDH

Track the analysis progress directly within the Developer Hub UI under the **MTA tab**.

![RHDH MTA Status](https://github.com/user-attachments/assets/3586774f-15f9-43e7-b683-4fe13a719fb2)

---

### 7. Confirm in MTA Web Console

Optionally, verify the same analysis from the **MTA UI** running on OpenShift.

![MTA UI Status](https://github.com/user-attachments/assets/5aba2bca-10b6-43c8-b880-5bcf5a3ea9ee)

---

### 8. Final Status in RHDH

After completion, return to the Developer Hub entity page to view final analysis output and recommendations.

![Final RHDH View](https://github.com/user-attachments/assets/195cc34f-6b54-49a2-9332-033ac371a936)

---





