
# How to Embed YieldX  Products

Embed YieldX products on your website to connect your users to modern fixed income portfolio construction and analysis tools.

## OVERVIEW

Embedded Products are client-side components embedded in your website that your users interact with in order to build and analyze fixed income portfolios.

With the click of a button on your website your users can start a flow.

## FLOW

<img width="789" alt="Embed YieldX Flow" src="https://user-images.githubusercontent.com/3128896/112641935-717f9000-8e08-11eb-8335-748f20dfacfd.png">

The flow begins when your user clicks to use an embedded product.
1. Your server makes a request to generate a session. The session will include a `sessionId` and user `accessToken`. See the [server-side integration](#server-side-integration) section for more details.
2. Place the `sessionId` and `accessToken` in the configuration object and then call `openYieldXApp(config)` on the UI. This will open the embedded product. See the [client-side integration](#client-side-integration) section for more details.
3. Upon user completion of the embedded product YieldX will send the results to your server via a webhook.

## SERVER SIDE INTEGRATION

### Authentication
When your institution is onboarded you will be proided an `institutionId`, `clientId`, and `clientSecret`.

To access our APIs, you will need a valid JSON Web Token (JWT). You can retrieve an `accessToken` with your `clientId` and `clientSecret` by calling our OAuth endpoint.

**OAuth Request:**
```json
POST https://auth.yieldx.app/oauth/token
{
    "client_id": "<<CLIENT_ID>>",
    "client_secret": "<<CLIENT_SECRET>>",
    "grant_type": "client_credentials",
    "audience": "https://yieldx.app"
}
```
**OAuth Response:**
```json
{
    "access_token": "<<ACCESS_TOKEN>>",
    "scope": "institution",
    "expires_in": 86400,
    "token_type": "Bearer"
}
```
Access tokens are valid for 24 hours. You should set up your systems to automatically update access tokens before they expire to maintain uninterrupted service. In the [coding examples](#coding-examples) section we provide an example backend solution for token management.

When calling YieldX APIs place the `accessToken` in the Authorization header of the API as a Bearer token. Additionally, you will need to place the `institutionId` in the API URL where it says `institutionId`.

### Generating Sessions
With your institution `accessToken` in hand you can now generate sessions. Create a server-side API that your website calls when a user clicks to open an embedded product. This API should call our [Generate Session](https://docs.yieldx.app/service.html?service=auth#operation/GenerateSession) API. Your API should return to the website the `sessionId` and session `accessToken`. Load the `sessionId` and `accessToken` into the embedded product configuration object and then call `openYieldXApp(config)`. See the [client-side integration](#client-side-integration) section for more details. 

IMPORTANT NOTES:
1. Keep you `clientSecret` in a safe place and do not share or expose it. If is does get exposed please reach out to us and we will generate a new secret.
2. Do not expose your institution `accessToken` to the client-side. This should only be used on the server-side.
3. The session `accessToken` is meant to be shared on the client-side.

### Configuring Webhooks
When a flow completes we provide the results to you via a webhook. You will need to build an API that can accept this data.
```json
{
    "sessionId": "string",
    "product": "INPAAS",
    "positions": [
        {
            "quantity": 10.0,
            "publicIdentifiers": {
                "cusip": "string",
                "isin": "string",
                "ticker": "string",
            }
        }
    ]
}
```

The product enum values are: "INPAAS", "BEST-FIT", and "ASSET-EXPLORER".

When your API is ready to recieve this data please reach out and we will configure your endpoint on our end.


## CLIENT SIDE INTEGRATION
You can embed YieldX products on your website by either installing our package or using a script tag.

### Installing the YieldX Package
You can use yarn or npm to install our package.
```sh
yarn add @yieldx/embed
npm install @yieldx/embed
```
Then import the package.
```jsx
import { openYieldXApp } from '@yieldx/embed'
```
### Using a script tag
You can retrieve the JavaScript file from our CDN.
```html
<script src="https://dp16xhm4dg09a.cloudfront.net/embed.esm.js"></script>
```
### Configuring and opening the app
```jsx
const onButtonClick = async () => { 

    /* Call your servers to generate a session */
    const server_side_api = https://your-domain.com/apis/generate-yieldx-session
    const session = await axios.get(server_side_api)
    const accessToken = session.data.accessToken
    const sessionId = session.data.sessionId

    /* Config YieldX object */
    const config = {
        "app": "inpaas",
        "container": "root",
        "access_token": accessToken,
        "session_id": sessionId,
        "theme": {
            "primary": "rgba(0,0,0,0)",
            "secondary": "rgba(0,0,0,0)",
            "foreground": "rgba(0,0,0,0)",
            "textPrimary": "rgba(0,0,0,0)",
            "textSecondary": "rgba(0,0,0,0)",
        }
    }

    /* Open YieldX App */
    openYieldXApp(config)

}
```
The app enum values are: "inpaas", "best-fit", and "asset-explorer"

### Adding themes
The theme property is optional. It allows you to customize colors to match your website. If no theme is provided the default theme will be used.
Please provide RGBA values so we can properly sanitize the values.
Please see below an illustration of how these colors are applied.

<img width="1585" alt="theme" src="https://user-images.githubusercontent.com/49527030/112526237-5a3c9600-8d67-11eb-8ee5-28df206e4aa3.png">


## CODING EXAMPLES
### How to automatically update access tokens and generate user sessions on the server-side
```jsx
import axios from "axios"
import jwt_decode from "jwt-decode"

// Institution static keys are provided by YieldX during onboarding.
const INSTITUTION_ID = "<<INSTITUTION ID>>"
const CLIENT_ID = "<<CLIENT ID>>"
const CLIENT_SECRET = "<<CLIENT SECRET>>"


const auth_url = "https://auth.yieldx.app/oauth/token"
const session_url = `https://dev.yieldx.app/apis/auth/v1/institutions/${INSTITUTION_ID}/sessions/_generate-session`

/*
This class provides example methods on how to authenticate with YieldX and generate customer sessions programmatically.
*/
class Authentication { 
    constructor(authToken) {
      this.authToken = null
    }  


    /* 
    This method returns a valid JWT that institutions use to authenticate with YieldX.
    The method will automatically renew the token before it expires.
    Call this method when an authToken is needed, provide the token in the Authorization header as a Bearer       token.
    For an example of how this is used see the generateYieldXSession method in this class.
    */
    getAuthToken = async () => {
        if (this.authToken == null || this.isTokenExpired(this.authToken)) {
          this.authToken = await this.generateYieldXToken()
        }
        return this.authToken
    }

    /*
    This method determines if the token is within a half hour of expiring.
    */
    isTokenExpired = (token) => {
        const expiresAt = jwt_decode(token).exp  
        const now = Date.now() / 1000
        return now > (expiresAt - 1800)
    }

    /*
    This method requests a new authToken from YieldX.
    */
    generateYieldXToken = async () => {
        const payload = {
            grant_type: "client_credentials",
            audience: "https://yieldx.app",
            client_id: CLIENT_ID,
            client_secret: CLIENT_SECRET
        }

        try {              
            const response = await axios.post(auth_url, payload)
            return response.data.access_token
        } catch (error){
            console.log(`Error: unable to generate a new authToken. \n ${error.message}`)
        }
    }
    
    /*
    This method generates a customer session. Sessions return a sessionId and customer authToken.
    These key must be provided in the embedded products for them to work. Sessions last for a half hour.
    Note: Only a customer authToken can be used in an embedded product, an institution authToken is forbidden.
    */
   generateYieldXSession = async (userId) => {
      const token = await this.getAuthToken()
      const headers = {headers:{"Authorization": `Bearer ${token}`}}
      const payload = {"userId": userId}

      try {
          const response = await axios.post(session_url, payload, headers)
          return response.data
      } catch (error){
          console.log(`Error: unable to generate a customer session. \n ${error.message}`)
      }
    }

}




/*
To run this class simply pull it into a function.
If you have NodeJS installed and would like to run this file execute these commands:
1) Add the INSITITUTION_ID, CLIENT_ID, and CLIENT_SECRET provided by YieldX.
2) npm install axios jwt-decode or yarn add axios jwt-decode
3) node yieldx-auth.js
*/
const runAuth = async () => {
  const authService = new Authentication()
  const authToken = await authService.getAuthToken()
  const session = await authService.generateYieldXSession()
  
  console.log(authToken)
  console.log(session)
}
runAuth()
```

## ASSISTANCE
If you need assitance please reach out to api-support@yieldx.app.

