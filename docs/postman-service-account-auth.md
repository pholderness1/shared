# Postman Service Account Auth

When making API calls against a Ping AIC tenant, you need to have an access token of some sort. The most common way to do this is with a service account, but that requires the use of a JWT bearer grant for the access token exchange.

Ping has [documented instructions](https://backstage.forgerock.com/docs/idcloud/latest/developer-docs/authenticate-to-rest-api-with-access-token.html) for how to get an access token, but these steps are not possible as part of a Postman request. Instead, we have to use the features available to us in Postman's script sandbox.

::: tip
If you just need an access token for a single request, the `frodo info` command will also print an access token as part of its output.
:::

## Generating and Signing a JWT

Although Postman does have support for some [external libraries](https://learning.postman.com/docs/tests-and-scripts/write-scripts/postman-sandbox-api-reference/#using-external-libraries), none of them have any functionality for RSA — the algorithm Ping AIC uses to sign JWT's.  Instead, we'll take advantage of a different option available to us, the [package library](https://learning.postman.com/docs/tests-and-scripts/write-scripts/package-library/).

The package library allows for us to create our own reusable pieces of code that can be called in pre-request and post-request scripts. To sign the JWT payload, I created my own library script that implements the RSA signing algorithm (specifically [RSASSA-PKCS1-v1_5](https://datatracker.ietf.org/doc/html/rfc3447#section-8.2)). The script is available in our Nexus repository as [postman-ping-aic-jwt.js](https://nexus.trivir.com/repository/postman-libs/postman-ping-aic-jwt.js).

::: details How do I add the script to my library?
See Postman's docs on [adding a new package](https://learning.postman.com/docs/tests-and-scripts/write-scripts/package-library/#add-a-new-package) to add the script to your own library.

*Postman does not support importing or uploading the script. You will need to copy and paste it in manually.*
:::

As most of the internals of the library script are not needed, it only exposes one `generateJwt` function with the following signature:

```js
/**
 * @param {string} audience Value to use in the audience of the JWT claim
 * @param {string} id ID of the service account
 * @param {string} jwkString JWK of the service account
 * @returns {string} The generated JWT to exchange for an access token
 */
generateJwt(audience, id, jwkString) 
```

Most commonly, I read the values that will be passed to the `generateJwt` function from my Postman Environment. For example:

```js
// base_url = 'https://openam.example.com/am'
// service_account_id = 'a842cc03-3334-43af-abbb-01eb3781414e'
// jwkString = '{"d":"6T2n8yu0rKHLDKHZRrqDJyAYordChVX...'

const audience = pm.environment.get('base_url') + '/oauth2/access_token';
const id = pm.environment.get('service_account_id');
const jwkString = pm.environment.get('service_account_jwk');

const jwt = generateJwt(audience, id, jwkString);
// eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIx...
```

With this freshly signed JWT, we now have everything we need to get an access token to use for all Ping AIC calls that require authorization.

## Exchanging the JWT for an Access Token

The JWT we generated alone is not enough authorization to make Ping AIC requests; we need to exchange it for an access token. This can be done by making a request to the AIC tenant's `/oauth2/access_token` endpoint, passing the JWT as the assertion for a [jwt-bearer](https://datatracker.ietf.org/doc/html/rfc7523) flow. For example with Postman:

```js
const authRequest = {
    url:  audience,
    method: 'POST',
    header: {
        'Accept': '*/*',
        'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: {
        mode: 'urlencoded',
        urlencoded : [
        { key: 'client_id', value: 'service-account' },
        { key: 'grant_type', value: 'urn:ietf:params:oauth:grant-type:jwt-bearer' },
        { key: 'assertion', value: jwt },
        { key: 'scope', value: 'fr:am:* fr:idm:*' },
        ]
    }
};
pm.sendRequest(authRequest, function (err, response) {
  const responseBody = response.json();
  // responseBody.access_token: eyJhbGciOiJIUzI1NiIsIn...
});
```

Exchanging and setting the access token manually in every request we make gets to be tedious and error-prone. Instead, it's better to configure a pre-request script that runs and assigns the access token to a variable that we can use in the request.

## Full Example

As a full example, this is the pre-request script that I set on my Ping AIC Postman Collection. If there is no access token saved or if the saved token is expired, then it generates a JWT. It then exchanges the JWT for an access token and saves the new token and its expiry date in my environment.

```js
const { generateJwt } = pm.require('postman-ping-aic-jwt');

function getAccessToken() {
    // Check for existing
    const accessToken = pm.environment.get('@auto_access_token');
    const expires = pm.environment.get('@auto_token_expires');
    if (accessToken && expires && Date.now() < expires) {
        return;
    }
    console.log('Getting new token');

    // Read environment
    const audience = pm.environment.replaceIn('https://{{host}}{{port}}{{am_path}}/oauth2/access_token');
    const id = pm.environment.get('service_account_id');
    const jwkString = pm.environment.get('service_account_jwk');

    // Use service account
    const jwt = generateJwt(audience, id, jwkString);

    // Exchange for access_token
    const authRequest = {
        url:  audience,
        method: 'POST',
        header: {
            'Accept': '*/*',
            'Content-Type': 'application/x-www-form-urlencoded',
        },
        body: {
            mode: 'urlencoded',
            urlencoded : [
            { key: 'client_id', value: 'service-account' },
            { key: 'grant_type', value: 'urn:ietf:params:oauth:grant-type:jwt-bearer' },
            { key: 'assertion', value: jwt },
            { key: 'scope', value: 'fr:am:* fr:idm:*' },
            ]
        }
    };
    pm.sendRequest(authRequest, function (err, response) {
        if (err) {
            throw err
        }
        if (response.code !== 200) {
            throw new Error(`Failed to get access token: [${response.code}]: ${response.text()}`)
        }
        const responseBody = response.json();
        pm.environment.set('@auto_access_token', responseBody.access_token);
        const expirationTime = Date.now() + responseBody.expires_in * 1000
        pm.environment.set('@auto_token_expires', expirationTime)
    });
}

getAccessToken();
```
