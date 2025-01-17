---
title: Registration Inline Hook Reference
excerpt: Customize handling of user registration requests in Profile Enrollment
---

# Registration Inline Hook Reference

This page provides reference documentation for:

- JSON objects contained in the outbound request from Okta to your external service

- JSON objects you can include in your response

This information is specific to the Registration Inline Hook, one type of Inline Hook supported by Okta.

## See also

For a general introduction to Okta Inline Hooks, see [Inline Hooks](/docs/concepts/inline-hooks/).

For information on the API for registering external service endpoints with Okta, see [Inline Hooks Management API](/docs/reference/api/inline-hooks/).

For steps to enable this Inline Hook, see [Enabling a Registration Inline Hook](#enable-a-registration-inline-hook-in-okta-identity-engine). <ApiLifecycle access="ie" /><br>

For an example implementation of this Inline Hook, see [Registration Inline Hook](/docs/guides/registration-inline-hook/nodejs/main/).

## About

The Okta Registration Inline Hook allows you to integrate your own custom code into Okta's [Profile Enrollment](https://help.okta.com/okta_help.htm?type=oie&id=ext-create-profile-enrollment) flow. The hook is triggered after Okta receives the registration request but before the user is created. Your custom code can:

- Set or override the values that will be populated in attributes of the user's Okta profile
- Allow or deny the registration attempt, based on your own validation of the information the user has submitted

> **Note:** Profile Enrollment and Registration Inline Hooks only work with the [Okta Sign-In Widget](/code/javascript/okta_sign-in_widget/) version 4.5 or later.

## Objects in the Request from Okta

The outbound call from Okta to your external service includes the following objects in its JSON payload:

### requestType

OTP request or event for which this transaction is being requested: self-service registration (SSR) or Progressive Enrollment.

Values for `requestType` are one of the following:

| Enum Value | Associated Okta Event |
|----------|-------------------------------------------------------|
| `self.service.registration` | self-service registration |
| `progressive.profile` | Progressive Enrollment |

### data.userProfile

This object appears in SSR requests from Okta. The object contains name-value pairs for each registration related attribute supplied by the user in the Profile Enrollment form, including:

- `lastName`
- `firstName`
- `login`
- `email`
- other custom attributes on the Sign-In Widget

The following attributes aren't included in the `data.userProfile` object:

- the `password` field
- any fields corresponding to user profile attributes marked as sensitive in your Okta user schema

Using the `com.okta.user.profile.update` commands you send in your response, you can modify the values of the attributes, or add other attributes, before the values are assigned to the Okta user profile that will be created for the registering user.

You can only set values for profile fields which already exist in your Okta user profile schema: Registration Inline Hook functionality can only set values; it cannot create new fields.

### data.UserProfileUpdate

<ApiLifecycle access="ie" /><br>

This object appears in Progressive Profile requests from Okta. The object contains the delta between existing name-value pairs and the attributes that your end user wants to update.

> **Note:** You can also allow end users to update non-sensitive attributes in addition to the delta attributes Okta sends in the request.

Use the `com.okta.user.progressive.profile.update` command in your response to progressively change the values of delta attributes in the user's Okta profile.

### data.action

> **Note:** The `data.action` object can appear in both SSR and Progressive Profile requests.

The action that Okta is currently set to take, regarding whether to allow this registration attempt.

There are two possible values:

- `ALLOW` indicates that the registration attempt will be allowed to proceed
- `DENY` indicates that the registration attempt will be terminated (no user will be created in Okta)

The action is `ALLOW` by default (in practice, `DENY` will never be sent to your external service).

Using the `com.okta.action.update` [command](#supported-commands) in your response, you can change the action that Okta will take.

### data.context.user

<ApiLifecycle access="ie" /><br>



## Response objects that you send

The objects that you can return in the JSON payload of your response are an array of one or more `commands`, to be executed by Okta, or an `error` object, to indicate problems with the registration request. These objects are defined as follows:

### commands

The `commands` object lets you invoke commands to modify or add values to the attributes in the Okta user profile that will be created for this user, as well as to control whether or not the registration attempt is allowed to proceed.

This object is an array, allowing you to send multiple commands in your response. Each array element requires a `type` property and a `value` property. The `type` property is where you specify which of the supported commands you wish to execute, and `value` is where you supply parameters for that command.

| Property | Description                                           | Data Type       |
|----------|-------------------------------------------------------|-----------------|
| type     | One of the [supported commands](#supported-commands). | String          |
| value    | Operand to pass to the command.                       | [value](#value) |

For example commands, see the [value](#value) section below.

#### Supported commands

The following commands are supported for the Registration Inline Hook type:

| Command                      | Description                                                  |
|------------------------------|--------------------------------------------------------------|
| com.okta.user.profile.update | Change attribute values in the user's Okta user profile. For SSR only. Invalid if used with a Progressive Profile response.  |
| com.okta.action.update       | Allow or deny the user's registration.                       |
| com.okta.user.progressive.profile.update   | Change attribute values in the user's Okta Progressive Profile. <ApiLifecycle access="ie" /> |

To set attributes in the user's Okta profile, supply a type property set to `com.okta.user.profile.update`, together with a `value` property set to a list of key-value pairs corresponding to the Okta user profile attributes you want to set. The attributes must already exist in your user profile schema.

To explicitly allow or deny registration to the user, supply a type property set to `com.okta.action.update`, together with a value property set to `{"registration": "ALLOW"}` or `{"registration": "DENY"}`. The default is to allow registration.

In Okta Identity Engine, to set attributes in the user's profile, supply a type property set to `com.okta.user.progressive.profile.update`, together with a `value` property set to a list of key-value pairs corresponding to the Progressive Enrollment attributes that you want to set. See [Registration Inline Hook - Send response](/docs/guides/registration-inline-hook/nodejs/main/#send-response). <ApiLifecycle access="ie" />

Commands are applied in the order that they appear in the array. Within a single `com.okta.user.profile.update` or `com.okta.user.progressive.profile.update` command, attributes are updated in the order that they appear in the `value` object.

You can never use a command to update the user's password, but you are allowed to set the values of attributes other than password that are designated sensitive in your Okta user schema. However, the values of those sensitive attributes, if included as fields in the Profile Enrollment form, aren't included in the `data.userProfile` object sent to your external service by Okta. See [data.userProfile](#data-userProfile).

#### value

The `value` object is the parameter to pass to the command.

For `com.okta.user.profile.update` commands, `value` should be an object containing one or more name-value pairs for the attributes you wish to update, for example:

```json
{
   "commands":[
      {
         "type":"com.okta.user.profile.update",
         "value":{
            "middleName":"Danger",
            "customerId":12345
         }
      }
   ]
}
```

The above example assumes that there is an attribute `customerId` defined in your Okta user schema (`middleName` is defined by default).

The same result could also be accomplished by means of two separate `com.okta.user.profile.update` commands, as follows:

```json
{
   "commands":[
      {
         "type":"com.okta.user.profile.update",
         "value":{
            "middleName":"Danger"
         }
      },
      {
         "type":"com.okta.user.profile.update",
         "value":{
            "customerId":12345
         }
      }
   ]
}
```

For `com.okta.action.update` commands, `value` should be an object containing the attribute `action` set to a value of either `ALLOW` or `DENY`, indicating whether the registration should be permitted or not, for example:

```json
{
  "commands": [
    {
      "type": "com.okta.action.update",
      "value": {
        "registration": "DENY"
      }
    }
  ]
}
```

Registrations are allowed by default, so setting a value of `ALLOW` for the `action` field is valid but superfluous.

### error

See [error](/docs/concepts/inline-hooks/#error) for general information about the structure to use for the `error` object.

For the Registration Inline Hook, the `error` object provides a way of displaying an error message to the end user who is trying to register.

* If you're using the Okta Sign-In Widget for Profile Enrollment, only the `errorSummary` messages of the `errorCauses` objects that your external service returns appear as inline errors, given the following:

   * You don't customize the error handling behavior of the widget.
   * The `location` of `errorSummary` in the `errorCauses` object specifies the request object's user profile attribute. See [JSON response payload objects - error](/docs/concepts/inline-hooks/#error).

* If you don't return a value for the `errorCauses` object, and deny the user's registration attempt through the `commands` object in your response to Okta, one of the following generic messages appears to the end user:</br></br>
      `Registration cannot be completed at this time.`</br></br>
      `We found some errors. Please review the form and make corrections.` <ApiLifecycle access="ie" />

* If you don't return an `error` object at all and the registration is denied, the following generic message appears to the end user:</br></br>
      `Registration denied.`</br></br>
      `Profile update denied.` <ApiLifecycle access="ie" />

> **Note:** If you include an error object in your response, no commands are executed and the registration fails. This holds true even if the top-level `errorSummary` and the `errorCauses` objects are omitted.

## Timeout behavior

If there is a response timeout after receiving the Okta request, the Okta process flow stops and registration is denied. The following message appears: "There was an error creating your account. Please try registering again".

## Sample JSON payload of request

> **Note:** The `requestType` field has a value of either `self.service.registration` or `progressive.profile`. See [requestType](#requesttype).

```json
{
  "eventId": "GOsk4z6tSSeZo6X08MvKaw",
  "eventTime": "2019-08-27T18:07:24.000Z",
  "eventType": "com.okta.user.pre-registration",
  "eventTypeVersion": "1.0",
  "contentType": "application/json",
  "cloudEventVersion": "0.1",
  "source": "reghawlks3zOkRrau0h7",
  "requestType": "progressive.profile",
  "data": {
    "context": {
      "request": {
        "id": "XWVxW2zcaH5-Ii74OsI6CgAACJw",
        "method": "POST",
        "url": {
          "value": "/api/v1/registration/reghawlks3zOkRrau0h7/register"
        },
        "ipAddress": "98.124.153.138"
      }
    },
    "userProfile": {
      "lastName": "Doe",
      "firstName": "John",
      "login": "john.doe@example.com",
      "email": "john.doe@example.com"
    },
    "action": null
  }
}
```

## Sample JSON payload of response

```json
{
  "commands": [
    {
      "type": "com.okta.action.update",
      "value": {
        "registration": "DENY"
      }
    }
  ],
  "error": {
    "errorSummary": "Incorrect email address. Please contact your admin.",
    "errorCauses": [
      {
        "errorSummary": "Only example.com emails can register.",
        "reason": "INVALID_EMAIL_DOMAIN",
        "locationType": "body",
        "location": "data.userProfile.login",
        "domain": "end-user"
      }
    ]
  }
}
```

## Enable a Registration Inline Hook in Okta Identity Engine

<ApiLifecycle access="ie" /><br>

To activate the Inline Hook, you first need to register your external service endpoint with Okta; see [Inline Hook Setup](/docs/concepts/inline-hooks/#inline-hooks_setup).

You must [enable and configure a Profile Enrollment policy](https://help.okta.com/okta_help.htm?type=oie&id=ext-create-profile-enrollment) to implement a Registration Inline Hook.

> **Note:** Profile Enrollment and Registration Inline Hooks are only supported by the [Okta Sign-In Widget](/docs/guides/embedded-siw/) version 4.5 or later.

To associate the Registration Inline Hook with your Profile Enrollment policy:

1. In the Admin Console, go to **Security > Profile Enrollment**.

1. Click the Pencil icon to edit the policy and associate it with your Registration Inline Hook.

1. In **Profile enrollment**, click **Edit**.

1. Select **Allowed** for **Self-service registration**.

1. From the **Inline hook** dropdown menu, select the hook that you set up and activated. See [Set up and activate the Registration Inline Hook](/docs/guides/registration-inline-hook/nodejs/main/#set-up-and-activate-the-registration-inline-hook).

   > **Note:** You can associate only one Inline Hook at a time with your Profile Enrollment policy.

1. In **Run this hook**, select when you want your Inline Hook to run:
   * **When a new user is created**: This trigger occurs during a self-service registration request.
   * **When attributes are collected for an existing user**: This trigger occurs during a Progressive Enrollment sign-in request.
   * **Both**: This trigger occurs during a self-service registration request and a Progressive Enrollment sign-in request.

1. Click **Save**.

Your Registration Inline Hook is configured for Profile Enrollment.

## Enable a Registration Inline Hook for SSR in Classic Engine

<ApiLifecycle access="deprecated" />

> **Note:** SSR in Okta Classic Engine is deprecated. See [Enable a Registration Inline Hook in Okta Identity Engine](#enable-a-registration-inline-hook-in-okta-identity-engine) for details.

To activate the Inline Hook, you first need to register your external service endpoint with Okta; see [Inline Hook Setup](/docs/concepts/inline-hooks/#inline-hooks_setup).

You then need to associate the registered Inline Hook with your Self-Service Registration policy. (For information on configuring a Self-Service Registration policy, see [Enable and configure a self-service registration policy](https://help.okta.com/okta_help.htm?id=ext_self_service_registration_policy).)

1. Go to **Directory > Self-Service Registration**.

1. Click **Edit**.

1. Select your hook from the **Extension** dropdown. If you have created multiple Registration Inline Hooks, you should see all of them displayed here.

1. Click **Save**.

Your Registration Inline Hook is now configured for Self-Service Registration.

> **Note:** Only one Inline Hook can be associated with your Self-Service Registration policy at a time.
