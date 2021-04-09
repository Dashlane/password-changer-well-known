# Password Changer .well-known [DRAFT]

The goal of this respository is to provide a documentation for using password changer .well-known and be listed as compatible on Dashlane's automatic password changer feature.

## Introduction

The password changer is able to do automatic password update by calling a compatible service's API endpoint.

In order to be compatible, a service needs:

-   a .well-known file placed at the web root of the service describing the API endpoint to call
-   an API endpoint that will receive the password change request

## Implementing the .well-known on your server

The following json file should be put on your web root at `/.well-known/password-changer`.

```js
{
    "version": "1.0",
    "endpoints": [
        // Multiple endpoints (if you need QA)
        {
            "auth": "Form",
            "url": "https://api.example.com/1.0/password_changer" // Must be https
        },
        {
            "auth": "Form",
            "url": "https://api.example.com/1.0/password_changer_qa", // Must be https
            "allowList": ["hashOfEmail1", "hashOfEmail1"] // SHA256 hash list of authorized emails
        }
    ]
}
```

The `auth` parameter defines how Password Changer will authenticate to the service API. Please, refer to the next section for the available auth options.

The `url` parameter defines the address of the API endpoint where password changer will send requests.

The `allowList` parameter defines a list of authorized emails that will be allowed to use a specific endpoint.
For instance, if you need to do some testing of a new API endpoint, you can set the hash (SHA256) of your Dashlane's account login in this list.

## API endpoint: Authentication

### Form Auth

Password Changer will make an HTTPS POST request to the specified API endpoint with the following structure:

```bash
curl --location --request POST 'https://api.example.com/api/changePassword' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'password=oldpassword' \
--data-urlencode 'login=user@mail.com' \
--data-urlencode 'newPassword=azerty12'

```

The 3 expected body fields are:

-   `username` (string)
-   `password` (string)
-   `newPassword` (string)

### Others Auth

For now we only support `Form` Auth but in the future we will support `Basic` Auth and `Digest` Auth.

## API endpoint: Replying to password change requests

In order to update the password in the user's vault or handle a potential error, password changer expect a JSON formated object with the following specifications.

All reply should be JSON object with a status key like this:

```json
{
    "status": "STATUS_CODE"
}
```

### Success

If the change is successful, you should reply a **200 response** with `OK` status.

```json
{
    "status": "OK"
}
```

### Errors

In case of an error, you should reply a **401 response** and we support the following error codes:

```json
{
    "LOGIN.PASSWORD_INCORRECT": {
        "description": "Incorrect user supplied password."
    },
    "LOGIN.NOT_FOUND": {
        "description": "Login supplied by user not present in site authentication database."
    },
    "LOGIN.GENERIC_FAILURE": {
        "description": "Incorrect credentials, login or password."
    },
    "LOGIN.ACCOUNT_LOCKED": {
        "description": "The user account is locked."
    },
    "SECURITY_REQUIREMENT.TOO_SHORT": {
        "description": "The new password is too short."
    },
    "SECURITY_REQUIREMENT.TOO_LONG": {
        "description": "The new password is too long."
    },
    "SECURITY_REQUIREMENT.CAN_NOT_REUSE_PREVIOUS_PASSWORD": {
        "description": "The new password must be different from the old password."
    },
    "SECURITY_REQUIREMENT.NO_SEQUENTIAL_CHARS": {
        "description": "The password can't contain a sequence of the same character."
    },
    "USER.PROFILE_INCOMPLETE": {
        "description": "The user must complete their profile before being able to change their password."
    },
    "USER.ACCOUNT_NOT_VERIFIED": {
        "description": "The user account is not verified. Typically a link sent by email during the registration has not been clicked."
    },
    "USER.NEEDS_TO_ACCEPT_TOS": {
        "description": "The user must accept Term Of Service."
    },
    "NEED_USER_ACTION": {
        "description": "There's something that prevent us to change user password (unpaid bill...)."
    },
    "WEBSITE_UNAVAILABLE": {
        "description": "The site is in maintenance, not accessible."
    },
    "SECURITY_REQUIREMENT.NOT_STRONG_ENOUGH": {
        "description": "New password rejected by site password policies."
    },
    "ABORTED": {
        "description": "Password change has been interrupted."
    },
    "VERIFICATION.METHOD_VERIFICATION_FAIL": {
        "description": "(NA)"
    },
    "VERIFICATION.WRONG_CODE": {
        "description": "Bad user challenge code."
    },
    "VERIFICATION.TIMEOUT": {
        "description": "The user challenge resolution is taking too long."
    },
    "VERIFICATION.UNKNOWN_VERIFICATION_ERROR": {
        "description": "Something bad happened while solving a user challenge."
    },
    "UNKNOWN_ERROR": {
        "description": "Something bad happened."
    }
}
```

### Captcha and 2FA

You should reply a **400 response** with the `NEED_VERIFICATION` status.

We will then ask the user to answer solve the challenge, and will call again your endpoint.
Two parameters, `verificationResponse` and optionally `verificationResponseKey`, will be sent alongside the other parameters.

Dashlane only supports two main methods of verification for now:

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

In the reply, the parameter verificationResponse will be the [response token](https://developers.google.com/recaptcha/docs/verify), usually called `g-recaptcha-response`.

This object reCaptchaVerification can also be in the well-known manifest, in case you always need a verified captcha.

## Ready to join?

When you have implemented it, you can get in touch with us by email and we'll add you website to the list : [dev-relationship (at) dashlane.com](mailto:dev-relationship@dashlane.com).

If you have any question on this process use the same email or open an issue on the repository.

## License

This document and associated materials are provided by Dashlane under [CC BY 4.0 License](https://creativecommons.org/licenses/by/4.0/).
