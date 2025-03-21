
# Guide: Configure and Test MTA Plugin in Red Hat Developer Hub (RHDH)

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
# Notice the v1beta3 version
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
# some metadata about the template itself
metadata:
  name: v1beta3-demo
  title: Onboard Application to MTA
  description: Onboard a new application to MTA
spec:
  owner: mta
  type: service
  # these are the steps which are rendered in the frontend with the form input
  parameters:
    - title: Fill in some steps
      required:
        - name
      properties:
        name:
          title: Name
          type: string
          description: Unique name of the application
          ui:autofocus: true
    - title: Choose a location
      properties:
        url:
          title: Repository URL
          type: string
        branch: 
          title: Branch
          type: string
          description: Repository branch to use
          default: main
        rootPath: 
          title: Root Path
          type: string
          description: Path to the root of the repository
          default: '.'

  # here's the steps that are executed in series in the scaffolder backend
  steps:
    - id: create-app
      name: Create Application
      action: mta:createApplication
      input:
        name: ${{ parameters.name }}
        url: ${{ parameters.url }}
        branch: ${{ parameters.branch }}
        rootPath: ${{ parameters.rootPath }}
  # some outputs which are saved along with the job for use in the frontend
  output:
    links:
      - title: Repository
        url: ${{ steps['publish'].output.remoteUrl }}
      - title: Open in catalog
        icon: catalog
        entityRef: ${{ steps['register'].output.entityRef }}

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

Go to **Catalog** and check that the registered template appears as an entity.

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





