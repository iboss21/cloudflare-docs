---
_build:
  publishResources: false
  render: never
  list: never
---

1. Log in to the [Cloudflare dashboard](https://dash.cloudflare.com/login) and select your account and domain.
2. Go to **Security** > **API Shield**.
3. Select **Settings**.
4. On **Endpoint settings**, select **Manage identifiers**.
5. Choose the type of session identifier (cookie, HTTP header, or JWT claim).

{{<Aside type="note">}}
If you are using a JWT claim, choose the [Token Configuration](/api-shield/security/jwt-validation/configure/#token-configurations) that will verify the JWT. Token Configurations are required to use JWT claims as session identifiers. 

Refer to [JWT Validation](/api-shield/security/jwt-validation/) for more information. 
{{</Aside>}}

6. Enter the name of the session identifier.
7. Select **Save**. 

After setting up session identifiers and allowing some time for Cloudflare to learn your traffic patterns, you can view your per endpoint and per session rate limiting recommendations, as well as enforce per endpoint and per session rate limits by creating new rules. Session identifiers will allow you to view API Discovery results from session ID-based discovery and session traffic patterns in Sequence Analytics.