# Use Cross Region Cognito as OIDC

Create a cognito in Standard AWS Region.

![cognito](./cognito.jpg)

After the cognito is created, launch the cloudformation.

### Get the OidcClientID
The OidcClientID is the App client ID

![app-client-id](./app-client-id.png)

### Get the OidcProvider
```
https://cognito-idp.${REGION}.amazonaws.com/${USER_POOL_ID}
```

![userpool-id](./userpool-id.png)

### CloudFormation parameters example

![example](./example.png)