# json-app

## Running in Anypoint
Import the file into anypoint and run as maven application

## Manually adding to Anypoint
Upload json-app.zip to anypoint studio

## Jenkins File (CI/CD)
#TODO

## Updating Environment
Reference links for modifying properties on running apps in Anypoint:
https://docs.mulesoft.com/mule-runtime/4.2/configuring-properties
https://docs.mulesoft.com/runtime-manager/anypoint-platform-cli-commands#runtime-mgr-cloudhub-application-modify

```plaintext
anypoint-cli runtime-mgr cloudhub-application modify --property mule.env:dev json-app
anypoint-cli runtime-mgr cloudhub-application modify --propertiesFile uat.properties json-app
```
