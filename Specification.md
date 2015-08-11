# Owlauth Specification

## Introduction

Owlauth is a new (and still experimental) protocol allowing applications to authenticate users without using traditional username / passwords.  This file documents details how the protocol works.  Other files will document the practicalities of using Owlauth as an application developer and as a provider of authentication services.

From a user's perspective the target Owlauth experience is:

1. Provide your email address to the application you wish to use.
1. See instructions telling you how to authenticate. 
1. Carrying out the action to authenticate (for example, by entering a PIN on a mobile application).
1. The application granting access from that point forward.

## Overview

Owlauth consists of four possible transactions:

1. Discovery.  Applications request an email address from the user (<user>@<domain>) and then looks for a JSON file at the URL `https://<domain>/.well-known/owlauth`.  This JSON file provides the address and port of the Owlauth server.
1. Login.  Applications POST the user's email address in a JSON object to the URL `https://<owlauthserver:port>/login`.  This results in login instructions for the user and a token.
1. Authentication.  Applications POST the token from Login to the url `https://<owlauthserver:port>/authenticate`.  The Owlauth server will authenticate the user (e.g. through email) and return to the application an auth token and validity period.
1. Refresh.  When an authtoken has been held for longer than the validity period, applications POST it to the URL `https://<owlauthserver:port>/refresh`.  The Owlauth server will provide a new token and validity period, or stipulate the application needs to carry out a new login.

##  Owlauth stages

### Discovery

To discover whether a user's organisation supports Owlauth, the application will carry out a GET on the URL `https://<domain>/.well-known/owlauth`.  The resulting JSON file has a single attribute `server` containing the location of the Owlauth server for this domain in the format `hostname:port` for example:

```
{
	"server": "example.com:3030"
}
```

### Login

Once the application has determine the location of the Owlauth server, it will then perform a POST on the URL `https://<owlauthserver:port>/login`, a JSON object with the following attribute:

 * `User` - The email address provided by the user.

The Owlauth server will return a JSON object with the following attributes:

 * `LoginText` - A string to present to the user telling them how to authenticate.
 * `LoginPhrase` - A string to present to the user to identify this authentication request from any others that are made.  Recommended to be a two common nouns, e.g. "Apple Bear"
 * `LoginToken` - An opaque string that must be presented to the Owlauth server in the following Authentication step.
 
 For example:

```
{
	"LoginText": "Please check your phone for an authentication request with this login phrase.",
	"LoginPhrase": "Apple Bear",
	"LoginToken": "A4BHJL25AM"
}
```

### Authentication

Immediately after completing the Login step, the application presents to the user both the LoginText and LoginPhrase.  Once these are shown to the user, and without any further action by the user, the application perfoms a POST on the url `https://<owlauthserver:port>/authenticate/` a JSON object with the following attribute:

 * `LoginToken` - The LoginToken from the Login step.
 
The Owlauth server will then carry out authentication of the user using one or more mechanisms.  The mechanism chosen for authentication is determined entirely by the domain owner and could include such examples as:

 * Emailing a one-time link to the user which, when followed, proves that they have access to their email account
 * Sending a notification to a pre-registered mobile application that requires a PIN to authenticate
 * Requesting the user to open an authentication application, protected by a traditional password mechanism, which displays authentication requests for approval

The result of the POST may take 5 minutes to return and the application should set timeouts accordingly.  If the POST returns an error of AUTH_TIMEOUT, or the HTTPS connection is lost, the application must start with a new Login request.  The Owlauth server may choose to re-issue the same LoginPhrase and LoginToken at its discretion.

A successful authentication will result in a JSON object with the following attributes:

 * `AuthenticatedToken` - An opaque string that can be presented to the Owlauth server for refresh once the validity period is over.
 * `ValidityDuration` - An integer containing the number of seconds that the authentication should be treated as valid for.  After this period is up a Refresh should be carried out.
 
 For example:

```
{
	"AuthenticatedToken": "ZCXVNASDF8945AS",
	"ValidityDuration": 3600
}
```

### Refresh

Once an application has held an AuthenticatedToken for longer than the ValidityDuration, it must carry out a Refresh in order to obtain confirmation that the authentication provided is still valid.  Doing this provides the Owlauth server the opportunity to force a new Login sequence, or to continue with no user interaction.

To do the refresh, the application perfoms a POST on the url `https://<owlauthserver:port>/refresh/` a JSON object with the following attribute:

 * `AuthenticatedToken` - The previously issued AuthenticatedToken to be refreshed.
 
A successful refresh will result in a new JSON object with new AuthenticatedToken and ValidityDuration values.

### Error Handling

In the event of an error the Login, Authenticate and Refresh end points will, instead of returning the values specified above, return a JSON object with the following attributes:

 * `ErrorCode` - A string containing equal to one of the defined error codes below.
 * `ErrorDescription` - A string containing a human readable explanation of the error.
 
 For example:

```
{
	"ErrorCode": "APPLICATION_NAME_TOO_LONG",
	"ErrorDescription": "The application name provided is too long."
}
```

The following table defines standard ErrorCode values and suggested application behaviour.

|ErrorCode|Description|Behaviour
----------|-----------|----------
|AUTH_TIMEOUT|The user failed to authenticate within the given timeout.|Restart the sequence with a new Login request.
|REFRESH_FAILED|The authentication token could not be refreshed and a new login sequence must be started.|Start a new Login sequence.
|AUTH_DECLINED|The user declined the authentication request.|Present an error to the user and take no further action without user input.
|UNABLE_TO_AUTHENTICATE|The server is unable to authenticate at this time, for example due to policies restricting the maximum number of requests outstandnig for a user.|Present error to user and provide option to start a new Login sequence.
|UNEXPECTED_INTERNAL_ERROR|The server encountered an unexpected error processing the request.|Present the error to the user and ask them to contact their domain provider.  Provide the option to start a new Login sequence.

