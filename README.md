# parse-server-api-mail-adapter <!-- omit in toc -->

[![npm version](https://badge.fury.io/js/parse-server-api-mail-adapter.svg)](https://badge.fury.io/js/parse-server-api-mail-adapter)
[![build status](https://github.com/mtrezza/parse-server-api-mail-adapter/workflows/ci/badge.svg?branch=main)](https://github.com/mtrezza/parse-server-api-mail-adapter/actions?query=workflow%3Aci+branch%3Amain)
[![codecov](https://codecov.io/gh/mtrezza/parse-server-api-mail-adapter/branch/main/graph/badge.svg)](https://codecov.io/gh/mtrezza/parse-server-api-mail-adapter)
[![vulnerabilities](https://snyk.io/test/github/mtrezza/parse-server-api-mail-adapter/badge.svg)](https://snyk.io/test/github/mtrezza/parse-server-api-mail-adapter)
[![dependency up-to-date](https://img.shields.io/librariesio/release/npm/parse-server-api-mail-adapter)](https://libraries.io/npm/parse-server-api-mail-adapter)
[![weekly downloads](https://img.shields.io/npm/dw/parse-server-api-mail-adapter)](https://www.npmjs.com/package/parse-server-api-mail-adapter)

The Parse Server API Mail Adapter enables Parse Server to send emails using any 3rd party API with built-in dynamic templates and localization.

# Content <!-- omit in toc -->

- [Installation](#installation)
- [Demo](#demo)
- [Configuration](#configuration)
- [Templates](#templates)
- [Placeholders](#placeholders)
    - [Password Reset and Email Verification](#password-reset-and-email-verification)
- [Localization](#localization)
- [Cloud Code](#cloud-code)
  - [Example](#example)
  - [Parameters](#parameters)
- [Need help?](#need-help)

# Installation

1. Install adapter:
    ```
    npm install --save parse-server-api-mail-adapter
    ```
2. Add [template files](#templates) to a subdirectory.
2. Add [adapter configuration](#configuration) to Parse Server.

# Demo

The demo script makes it easy to test adapter configurations and templates by sending emails without Parse Server via the email service provider [Mailgun](https://www.mailgun.com):

1. Create a file `mailgun.json` in the `demo` directory with the following content:
    ```js
    {
        "key": "MAILGUN_API_KEY", // e.g. abc123
        "domain": "MAILGUN_DOMAIN", // e.g. sandbox-example@mailgun.org
        "sender": "SENDER_EMAIL", // e.g. sender@example.com
        "recipient": "RECIPIENT_EMAIL" // e.g. recipient@example.com
    }
    ```
2. Run `node ./demo` to execute the script and send an email.

You can modify the script to use any other API you like or debug-step through the sending process to better understand the adapter internals.

# Configuration

An example configuration to add the API Mail Adapter to Parse Server could look like this:

```js
// Declare a mail client
const mailgun = require('mailgun.js');
const mailgunClient = mailgun.client({ username: 'api', key: process.env.MAILGUN_API_KEY });
const mailgunDomain = process.env.MAILGUN_DOMAIN;

// Configure Parse Server
const server = new ParseServer({
    ...otherOptions,

    emailAdapter: {
        module: 'parse-server-api-mail-adapter',
        options: {
            // The email address from which emails are sent.
            sender: 'sender@example.com',
            // The email templates.
            templates: {
                // The template used by Parse Server to send an email for password
                // reset; this is a reserved template name.
                passwordResetEmail: {
                    subjectPath: './files/password_reset_email_subject.txt'),
                    textPath: './files/password_reset_email.txt'),
                    htmlPath: './files/password_reset_email.html')
                },
                // The template used by Parse Server to send an email for email
                // address verification; this is a reserved template name.
                verificationEmail: {
                    subjectPath: './files/verification_email_subject.txt'),
                    textPath: './files/verification_email.txt'),
                    htmlPath: './files/verification_email.html')
                },
                // A custom email template that can be used when sending emails
                // from Cloud Code; the template name can be choosen freely; it
                // is possible to add various custom templates.
                customEmail: {
                    subjectPath: './files/custom_email_subject.txt'),
                    textPath: './files/custom_email.txt'),
                    htmlPath: './files/custom_email.html'),
                    // Placeholders are filled into the template file contents.
                    // For example, the placeholder `{{appName}}` in the email
                    // will be replaced the value defined here.
                    placeholders: {
                        appName: "ExampleApp"
                    },
                    // Extras to add to the email payload that is accessible in the
                    // `apiCallback`.
                    extra: {
                        replyTo: 'no-reply@example.com'
                    },
                    // A callback that makes the Parse User accessible and allows
                    // to return user-customized placeholders that will override
                    // the default template placeholders. It also makes the user
                    // locale accessible, if it was returned by the `localeCallback`,
                    // and the current placeholders that will be augmented.
                    placeholderCallback: async ({ user, locale, placeholders }) => {
                        return {
                            phone: user.get('phone');
                        };
                    },
                    // A callback that makes the Parse User accessible and allows
                    // to return the locale of the user for template localization.
                    localeCallback: async (user) => {
                        return user.get('locale');
                    }
                }
            },
            // The asynronous callback that contains the composed email payload to
            // be passed on to an 3rd party API and optional meta data. The payload
            // may need to be converted specifically for the API; conversion for
            // common APIs is conveniently available in the `ApiPayloadConverter`.
            // Below is an example for the Mailgun client.
            apiCallback: async ({ payload, locale }) => {
                const mailgunPayload = ApiPayloadConverter.mailgun(payload);
                await mailgunClient.messages.create(mailgunDomain, mailgunPayload);
            }
        }
    }
});
```

# Templates

Emails are composed using templates. A template defines the paths to its content files, for example:

```js
templates: {
    exampleTemplate: {
        subjectPath: './files/custom_email_subject.txt'),
        textPath: './files/custom_email.txt'),
        htmlPath: './files/custom_email.html'),
    }
},
```

There are different files for different parts of the email:
- subject (`subjectPath`)
- plain-text content (`textPath`)
- HTML content (`htmlPath`)

See the [templates](https://github.com/mtrezza/parse-server-api-mail-adapter/tree/main/spec/templates) for examples how placeholders can be used.

# Placeholders
Placeholders allow to dynamically insert text into the template content. The placeholder values are filled in according to the key-value definitions returned by the placeholder callback in the adapter configuration.

This is using the [mustache](http://mustache.github.io/mustache.5.html) template syntax. The most commonly used tags are:
- `{{double-mustache}}`: The most basic form of tag; inserts text as HTML escaped by default.
- `{{{triple-mustache}}}`: Inserts text with unescaped HTML, which is required to insert a URL for example.

### Password Reset and Email Verification

By default, the following placeholders are available in the password reset and email verification templates:
- `{{appName}}`: The app name as set in the Parse Server configuration.
- `{{username}}`: The username of the user who requested the email.
- `{{link}}`: The URL to the Parse Server endpoint for password reset or email verification.

# Localization

Localization allows to use a specific template depending on the user locale. To turn on localization for a template, add a `localeCallback` to the template configuration.

The locale returned by `localeCallback` will be used to look for locale-specific template files. If the callback returns an invalid locale or nothing at all (`undefined`), localization will be ignored and the default files will be used.

The locale-specific files are placed in subfolders with the name of either the whole locale (e.g. `de-AT`), or only the language (e.g. `de`). The locale has to be in format `[language]-[country]` as specified in [IETF BCP 47](https://tools.ietf.org/html/bcp47), e.g. `de-AT`.

Localized files are placed in subfolders of the given path, for example:
```js
base/
├── example.html         // default file
└── de/                  // de language folder
│   └── example.html     // de localized file
└── de-AT/               // de-AT locale folder
│   └── example.html     // de-AT localized file
```

Files are matched with the user locale in the following order:
1. **Locale** (locale `de-AT` matches file in folder `de-AT`)
2. **Language** (locale `de-AT` matches file in folder `de` if there is no file in folder `de-AT`)
3. **Default** (default file in base folder is returned if there is no file in folders `de-AT` and `de`)

# Cloud Code

Sending an email directly from Cloud Code is possible since Parse Server > 4.5.0. This adapter supports this convenience method.

## Example

If the `user` provided has an email address set, it is not necessary to set a `recipient` because the mail adapter will by default use the mail address of the `user`.

```js
Parse.Cloud.sendEmail({
  templateName: "next_level_email",
  placeholders: { gameScore: 100, nextLevel: 2 },
  user: parseUser // user with email address
});
```

## Parameters

| Parameter      | Type         | Optional | Default Value | Example Value               | Description                                                                                    |
|----------------|--------------|----------|---------------|-----------------------------|------------------------------------------------------------------------------------------------|
| `sender`       | `String`     |          | -             | `from@example.com`          | The email sender address; overrides the sender address specified in the adapter configuration. |
| `recipient`    | `String`     |          | -             | `to@example.com`            | The email recipient; if set overrides the email address of the `user`.                         |
| `subject`      | `String`     |          | -             | `Welcome`                   | The email subject.                                                                             |
| `text`         | `String`     |          | -             | `Thank you for signing up!` | The plain-text email content.                                                                  |
| `html`         | `String`     | yes      | `undefined`   | `<html>...</html>`          | The HTML email content.                                                                        |
| `templateName` | `String`     | yes      | `undefined`   | `customTemplate`            | The template name.                                                                             |
| `placeholders` | `Object`     | yes      | `{}`          | `{ key: value }`            | The template placeholders.                                                                     |
| `extra`        | `Object`     | yes      | `{}`          | `{ key: value }`            | Any additional variables to pass to the mail provider API.                                     |
| `user`         | `Parse.User` | yes      | `undefined`   | -                           | The Parse User that the is the recipient of the email.                                         |



# Need help?

- Ask on StackOverflow using the [parse-server](https://stackoverflow.com/questions/tagged/parse-server) tag.
- Search through existing [issues](https://github.com/mtrezza/parse-server-api-mail-adapter/issues) or open a new issue.