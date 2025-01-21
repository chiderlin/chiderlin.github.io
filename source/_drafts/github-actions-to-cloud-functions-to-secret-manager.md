---
title: github-actions-to-cloud-functions-to-secret-manager
date: 2024-08-14 14:13:21
tags:
categories:
---

secret_manager

```javascript
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');
const { GoogleAuth } = require('google-auth-library');

const auth = new GoogleAuth({
  keyFilename: 'credentials.json',
  scopes: 'https://www.googleapis.com/auth/cloud-platform',
});

project_id = 'projects/501682557119/secrets/googleSheetSecret/versions/latest'; // Project for which to manage secrets.
secretId = 'googleSheetSecret';

async function accessSecret() {
  const client = new SecretManagerServiceClient({ auth });
  // Access the secret.
  const [version] = await client.accessSecretVersion({
    name: project_id,
  });

  const responsePayload = version.payload.data.toString('utf8');
  const secretConfig = JSON.parse(responsePayload);
  // console.info(`Payload: ${responsePayload}`);
  return secretConfig;
}

module.exports = accessSecret;
```
