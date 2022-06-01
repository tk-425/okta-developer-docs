### 1 - 4: Sign in and Select Authenticator

The challenge flow follows the same first four steps as the [enrollment flow](/docs/guides/authenticators-web-authn/aspnet/main/#integrate-sdk-for-authenticator-enrollment):

1. Build a sign-in page on the client.
2. Authenticate the user credentials.
3. Handle the response from the sign-in flow.
4. Display a list of possible authenticator factors.

### 5: Retrieve encrypted challenge and user information

When the user selects the WebAuthn Authenticator factor and clicks **Submit**, the form posts back to the `SelectAuthenticatorAsync` method. This checks whether the user is in Challenge Flow or Enrollment Flow.

When in Challenge Flow, a call is made to `idxClient.SelectChallengeAuthenticatorAsync`, using its `selectAuthenticatorOptions` parameter to pass in the WebAuthn authenticator factor ID.

```csharp
var selectAuthenticatorOptions = new SelectAuthenticatorOptions
{
    AuthenticatorId = model.AuthenticatorId,
};

selectAuthenticatorResponse = await _idxClient.SelectChallengeAuthenticatorAsync(
    selectAuthenticatorOptions, (IIdxContext)Session["IdxContext"]);
```

If the call is successful, the returned `selectAuthenticatorResponse` object has an `AuthenticationStatus` of `AwaitingAuthenticatorVerification` and its `CurrentAuthenticator` property contains the challenge and other information to verify the WebAuthn credentials on the user’s device.

```csharp
Session["IdxContext"] = selectAuthenticatorResponse.IdxContext;

switch (selectAuthenticatorResponse?.AuthenticationStatus)
{
    case AuthenticationStatus.AwaitingAuthenticatorVerification:
        var action = (model.IsWebAuthnSelected)
            ? "VerifyWebAuthnAuthenticator"
            : "VerifyAuthenticator";
        if (model.IsWebAuthnSelected)
        {
            Session["currentWebAuthnAuthenticator"] =
                selectAuthenticatorResponse.CurrentAuthenticatorEnrollment;
        }
        return RedirectToAction(action, "Manage");

// other case statements

    default:
        return View("SelectAuthenticator", model);
}
```

### 6: Display challenge page

Build a page that takes the challenge and user information from Okta servers and calls `navigator.credentials.get` to raise a challenge prompt from the correct WebAuthn authenticator.

For example, in the sample application, a new `VerifyWebAuthnViewModel` is populated from the `CurrentAuthenticatorEnrollment` property of the `AuthenticatorResponse` object returned in the previous step.

```csharp
var currentAuthenticator = (IAuthenticator)Session["currentWebAuthnAuthenticator"];

var viewModel = new VerifyWebAuthnViewModel
{
    WebAuthnCredentialId = currentAuthenticator.CredentialId,
    Challenge = currentAuthenticator.ContextualData.ChallengeData.Challenge,
};
```

The `viewModel` parameter is then consumed in the script section of a Razor page. First it's prepared to be passed to the authenticator.

```js
const challenge = '@Model.Challenge';
const webauthnCredentialId = '@Model.WebAuthnCredentialId';
const publicKeyCredentialRequestOptions = {
    challenge: strToBin(challenge),
    allowCredentials: [
        {
            id: strToBin(webauthnCredentialId),
            type: 'public-key',
        }
    ],
    userVerification: 'discouraged',
    timeout: 60000,
};
```

The call to `navigator.credentials.get` calls WebAuthn APIs in the browser and passes the challenge to the authenticator for validation. The authenticator looks up the information stored for the credential ID and checks that the domain name matches the one that was used during enrollment.

If the validations are successful, the user sees the authenticator challenge dialog.

<div class="common-image-format">

![The authenticator challenge dialog](/img/authenticators/dotnet-authenticators-webauthn-challenge-prompt.png)

</div>

After the authenticator validates the user, it returns an assertion object that proves the user is who they say they are.

* `clientDataJSON`: A collection of the data passed from the browser to the authenticator
* `authenticatorData`: Some additional data from the authenticator
* `signature`: The signature generated by the private key in the authenticator

```js
    navigator.credentials.get({
            publicKey: publicKeyCredentialRequestOptions
        })
        .then((assertion) => {
            const clientData = binToStr(assertion.response.clientDataJSON);
            const authenticatorData = binToStr(assertion.response.authenticatorData);
            const signatureData = binToStr(assertion.response.signature);

            const params = {
                "clientData": clientData,
                "authenticatorData": authenticatorData,
                "signatureData": signatureData
            };

            const options = {
                method: 'POST',
                body: JSON.stringify(params),
                headers: { "Content-type": "application/json; charset=UTF-8" }
            };
```

### 7: Send signature for validation

Send the validating credentials back to the Okta server to finish validating the authentication data for the user. Okta decrypts the signature using the user's public key it stored during enrollment and validates that it's the same challenge that started the flow.

```js
            fetch('@Url.Action("VerifyWebAuthnAuthenticatorAsync", "Manage")', options)
                .then(res => {
                    console.log("Request successful! Response:", res);
                    location.href = '@Url.Action("VerifyWebAuthnAuthenticator",
                        "Manage", new { verificationCompleted = true })';
                })
                .catch(function(err) {
                    console.error(err);
                });
        })
        .catch(function (err) {
            console.error(err);
        });
}, false);
```

Pass the signature and other information as parameters to the `ChallengeAuthenticatorAsync` method on the `IdxClient`.

```csharp
var authnResponse = await _idxClient.ChallengeAuthenticatorAsync(
    new ChallengeWebAuthnAuthenticatorOptions
    {
        AuthenticatorData = viewModel.AuthenticatorData,
        ClientData = viewModel.ClientData,
        SignatureData = viewModel.SignatureData,
    }, (IIdxContext)Session["idxContext"]);

Session["webAuthnResponse"] = authnResponse;
```

### 8. Confirm successful verification and sign user in

Query the `AuthenticationStatus` property of the `AuthenticationResponse` object returned by `ChallengeAuthenticatorAsync` to discover the current status of the authentication process. A status of `Success` means that the user is now successfully signed in to the app. Call `AuthenticationHelper.GetIdentityFromTokenResponseAsync` to retrieve the OIDC claims information about the user and pass them into your application.

```csharp
 var authnResponse = (IAuthenticationResponse)Session["webAuthnResponse"];

switch (authnResponse?.AuthenticationStatus)
{
    case AuthenticationStatus.Success:
        ClaimsIdentity identity =
            await AuthenticationHelper.GetIdentityFromTokenResponseAsync(
                _idxClient.Configuration, authnResponse.TokenInfo);
        _authenticationManager.SignIn(identity);
        return RedirectToAction("Index", "Home");

     default:
        return RedirectToAction("Index", "Home");
}
```