### 1. Initiate sign in

First, the user initiates the sign-in with username and password by making a call to `OktaAuth.idx.authenticate()`.

```javascript
  const authClient = getAuthClient(req);
  const transaction = await authClient.idx.authenticate({ username,password});
```

### 2. Send email to user's email address

If the user is already enrolled in Okta email, calling `OktaAuth.idx.authenticate()` sends an email to the user's email address. This email, which contains a placeholder for the OTP, is based on the **Email Challenge** template.

### 3. Display OTP input page

`OktaAuth.idx.authenticate()` returns a response indicating the next step is to challenge the user with the email authenticator.  `IdxTransaction` returns with a `status` of `PENDING`, `nextStep.name` of `challenge-authenticator`, and `nextstep.authenticatorEnrollments[n],key` equal to `okta_email`.

```json
{
  status: "PENDING",
  nextStep: {
    name: "challenge-authenticator",
    inputs: [
      {
        name: "verificationCode",
        label: "Enter code",
        type: "string",
      },
    ],
    type: "email",
    authenticatorEnrollments: [
      {
        profile: {
          email: "johndoe@gmail.com",
        },
        type: "email",
        key: "okta_email",
        displayName: "Email",
      },
    ],
  },
}
```

Using this response, display a page to input the OTP. Although this use case covers the magic link scenario, displaying an OTP page allows for an OTP verification fallback in cases where OTP may be required or simply more convenient. For example, a user checking their email from a different device must use OTP. [Integrate different browser and device scenario](#integrate-different-browser-and-device-scenario) covers the integration details for the different browser and device scenarios.


<div class="common-image-format">

![Screenshot of OTP in challenge page](/img/authenticators/authenticators-email-challenge-auth.png)

</div>

### 4. Click on the email magic link

Next, the user opens their email and clicks on the magic link. The following screenshot shows the magic link in the email.

<div class="common-image-format">

![Magic link in email](/img/authenticators/authenticators-email-challenge-magic-link-in-email.png)

</div>

The link points to your Okta org as in: `https://yourorg.okta.com/email/verify/0oai9ifvveyL3QZ8K696?token=ftr2eAgsg...`

### 5. Send the request to Okta

When the user clicks on the magic link, your org receives the request, gets the `OTP` and `state` parameters, and forwards the request with these parameters to your application. The org combines the `Callback URI` you defined in [Update configurations](#update-configurations) with the `OTP` and `state` parameters to produce a final callback URL for the user. For example, `http://localhost:8080/login/callback?otp=726009&state=1b34371af02dd31d2bc4c48a3607cd32`

### 6: Handle redirect in your app

Create a route to handle the magic link redirect request.

```javascript
router.get('/login/callback', async (req, res, next) => {
  const { protocol, originalUrl } = req;
  const parsedUrl = new URL(protocol + '://' + req.get('host') + originalUrl);
});

```

### 7. Validate request is from a verification email

Call `OktaAuth.idx.isEmailVerifyCallback()` passing in the query parameter section of the URL. This method validates that the OTP and state query parameters are present. For example, `http://localhost:8080/login/callback?otp=726009&state=1b34371af02dd31d2bc4c48a3607cd32` will present the `otp` value `726009` and the `state` value `1b34371af02dd31d2bc4c48a3607cd32`

```javascript
    if (authClient.idx.isEmailVerifyCallback(search)) {
      //continue
    }
```

### 8. Verify the email and location of the magic link click

Next call `OktaAuth.idx.handleEmailVerifyCallback()` passing in the query parameter containing the OTP and state. This method performs the following checks:

* Ensures that the user clicked the magic link in the same browser (but different tab) as your application.
* Validates that the `otp` and `state` parameters originate from a nonexpired verification email


```javascript
  try {
      const transaction = await authClient.idx.handleEmailVerifyCallback(search);
      handleTransaction({ req, res, next, authClient, transaction });
  } catch (err) {
    next(err);
  }

```

### 9: Complete successful sign-in

`IdxTransaction` returns a status of `SUCCESS` along with access and ID tokens. Your app redirects the user to the default home page for the signed-in user.

```json
{
  status: "SUCCESS",
  tokens: {
    accessToken: {
      accessToken: "eyJraWQiOiJTajV3...",
      },
      expiresAt: 1646673230,
    },
    idToken: {
      idToken: "eyJraWQiOiJTajV3cUUwO...",
      expiresAt: 1646673230,
    },
  }
}

```