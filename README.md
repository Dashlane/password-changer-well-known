# Password Changer .well-known [DRAFT]

The goal of this respository is to create a specification for a [well-known resource](https://tools.ietf.org/html/rfc5785) to facilitate password managers to change the service password, which will be compatible with Dashlane Password Changer.

## Introduction

Dashlane Password Changer provides the ability to automatically update passwords by calling a compatible API endpoint.

In order to be compatible, a service must:

-   host a specific file within the `.well-known` folder at the web root of the service describing the API endpoint to call
-   host an API endpoint that will receive the password change request

## Implementing the .well-known description for password changer

The following JSON file should be placed at the root of your website: `/.well-known/password-changer`.

```js
{
    "version": "1.0",
    "endpoints": [
        // Multiple endpoints, allowing for a test endpoint
        {
            "auth": "Form",
            "url": "https://api.example.com/1.0/password_changer" // Must be https
        },
        {
            "auth": "Form",
            "url": "https://api.example.com/1.0/password_changer_qa", // Must be https
            "allowList": ["hashOfEmail1", "hashOfEmail1"] // SHA256 hash list of authorized logins
        }
    ]
}
```

The `auth` parameter defines the authenticattion method for the API endpoint. Please refer to the next section for the authentication methods currently supported by Dashlane Password Changer.

The `url` parameter defines the address of the API endpoint that implements the changing of the service password, which can be used by password managers.

The `allowList` parameter defines a list of authorized logins that will be allowed to use a specific endpoint.
For instance, if you need to do some testing of a new API endpoint, you can set the hash (SHA256) of your Dashlane's account login in this list.

## API endpoint: Authentication

### Form Auth

Password managers will make a HTTPS POST request to the specified API endpoint with the following structure:

```bash
curl --location --request POST 'https://api.example.com/api/changePassword' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'password=oldpassword' \
--data-urlencode 'login=user@mail.com' \
--data-urlencode 'newPassword=Correct Horse Battery Staple'

```

The 3 expected body fields are:

-   `username` (string)
-   `password` (string)
-   `newPassword` (string)

### Authentication supported by Dashlane Password Changer

Dashlane Password Changer currently only supports `Form` authentication. `Basic` and `Digest` authentication will be supported in future.

## API endpoint: Responding to password change requests

In order to confirm a password change has been accepted or to handle a potential error, the password changer endpoint should only respond with the following JSON object specification:

```json
{
    "status": "STATUS_CODE"
}
```

### Success

If the change is successful, you should respond with a **200 response** with the status set to `OK`.

```json
{
    "status": "OK"
}
```

### Errors

In case of an error, you should respond with a **401 response**, Dashlane currently supports the following errors:

| **Error code**                                       | **Description**                                                                                                |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| LOGIN.PASSWORD_INCORRECT                             | Incorrect user password submitted.                                                                              |
| LOGIN.NOT_FOUND                                      | Login unrecognized by the service.                                            |
| LOGIN.GENERIC_FAILURE                                | Login or password are incorrrect.                                                                      |
| LOGIN.ACCOUNT_LOCKED                                 | The account is locked.                                                                                    |
| SECURITY_REQUIREMENT.TOO_SHORT                       | The new password is too short.                                                                                 |
| SECURITY_REQUIREMENT.TOO_LONG                        | The new password is too long.                                                                                  |
| SECURITY_REQUIREMENT.CAN_NOT_REUSE_PREVIOUS_PASSWORD | The new password cannot be the same as a previously used password.                                                      |
| SECURITY_REQUIREMENT.NO_SEQUENTIAL_CHARS             | The password cannot contain a sequence of the same character.                                                   |
| USER.PROFILE_INCOMPLETE                              | The user must complete their profile before being able to change their password.                               |
| USER.ACCOUNT_NOT_VERIFIED                            | The user account is not verified. For example an email confirmation has not been actioned. |
| USER.NEEDS_TO_ACCEPT_TOS                             | The user must accept the Terms of Service.                                                                          |
| NEED_USER_ACTION                                     | A required user action is preventing the change of password (for example an unpaid bill).                                    |
| WEBSITE_UNAVAILABLE                                  | The site is currently undergoing maintaintance and is not accessible.                                                                    |
| SECURITY_REQUIREMENT.NOT_STRONG_ENOUGH               | The new password is rejected by site's password policy.                                                               |
| ABORTED                                              | The password change has been interrupted.                                                                          |
| VERIFICATION.METHOD_VERIFICATION_FAIL                | (NA)                                                                                                           |
| VERIFICATION.WRONG_CODE                              | Bad user challenge code.                                                                                       |
| VERIFICATION.TIMEOUT                                 | The user challenge resolution is taking too long.                                                              |
| VERIFICATION.UNKNOWN_VERIFICATION_ERROR              | Something went wrong while solving a user challenge.                                                         |
| UNKNOWN_ERROR                                        | Something went wrong.                                                                                        |

Here is an example response when the provided password is incorrect:

```json
{
    "status": "LOGIN.PASSWORD_INCORRECT"
}
```

### Captcha and 2FA

If the user is required to perform an intermediate action such as a Captha or a 2FA code verification, a response with a **400 response** with the status set to `NEED_VERIFICATION` should be made.

The password manager will then request the user to complete the challenge, after which the endpoint will be called again.

Two parameters, `verificationResponse` and optionally `verificationResponseKey`, will be sent alongside the regular parameters.

Currently two verification methods are supported by Dashlane:

1. `2faVerification`:
   Reply should be the following :

    ```js
    {
        "status": "NEED_VERIFICATION",
        "verificationType": "2FA",
        "2faVerification": {
            // This hint will be only shown when the user clicks on the "more info" button
            "hintText": "Enter the code we sent to your number ending in 99",
            "type": "SMS", // Optional. Can be SMS, EMAIL, APP, OTHER.
            "inputType": "DIGITS", // Optional. Can be DIGITS, LETTERS or ANY
            "inputLength": 4, // Optional
            "responseKey": "jkf3kf32ewfji32ijgger" // Optional. We can send this key back with the user response
        }
    }
    ```

2. `reCaptchaVerification`:

    ```js
    {
        "status": "NEED_VERIFICATION",
        "verificationType": "RECAPTCHA_V2",
        "reCaptchaVerification": {
            "sitekey": "6Le-wvkSAAAAAPBMRTvw0Q4Muexq9bi0DJwx_mJ-", // Sitekey, provided by recaptcha
            "domain": "dashlane-recaptcha.com", // Optional. The domain must be whitelisted in your recaptcha interface
            "responseKey": "jkf3kf32ewfji32ijgger" // Optional. We can send this key back with the user response
        }
    }
    ```

In the response, the parameter `verificationResponse` will be the [response token](https://developers.google.com/recaptcha/docs/verify), usually called `g-recaptcha-response`.

This object `reCaptchaVerification` can also be in the well-known manifest, in case you always need a verified captcha.

## Ready to join?

Once you have implemented the above please get in touch with us by email and we'll add your website to the list : [dev-relationship (at) dashlane.com](mailto:dev-relationship@dashlane.com).

If you have any question on this process use the same email or open an issue on the repository.

## License

This document and associated materials are provided by Dashlane under [CC BY 4.0 License](https://creativecommons.org/licenses/by/4.0/).
