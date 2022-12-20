# PHP Exchange Web Services

The PHP Exchange Web Services library (php-ews) is intended to make
communication with Microsoft Exchange servers using Exchange Web Services
easier. It handles the NTLM authentication required to use the SOAP
services and provides an object-oriented interface to the complex types
required to form a request.

[![Scrutinizer](https://img.shields.io/scrutinizer/g/jamesiarmes/php-ews.svg?style=flat-square)][1]
[![Total Downloads](https://img.shields.io/packagist/dt/php-ews/php-ews.svg?style=flat-square)][2]

## Dependencies

* Composer
* PHP 5.4 or greater
* cURL with NTLM support (7.30.0+ recommended)
* Exchange 2007 or later

**Note: Not all operations or request elements are supported on all versions of
Exchange.**

## Installation

The prefered installation method is via Composer, which will automatically
handle autoloading of classes.

```json
{
    "require": {
        "php-ews/php-ews": "~1.0"
    }
}
```

## Usage

The library can be used to make several different request types. In order to
make a request, you need to instantiate a new `\jamesiarmes\PhpEws\Client`
object:

```php
use \jamesiarmes\PhpEws\Client;

$ews = new Client($server, $version);
$ews->authWithOauth2($accesskey);
$ews->authWithUserAndPass($username, $password);
```

The `Client` class takes two parameters for its constructor:

* `$server`: The url to the exchange server you wish to connect to, without
  the protocol. Example: mail.example.com. If you have trouble determining the
  correct url, you could try using [autodiscovery][3].
* `$version` (optional): The version of the Exchange sever to connect to. Valid
  values can be found at `\jamesiarmes\PhpEws\Client::VERSION_*`. Defaults to
  Exchange 2007.

The `authWithOauth2` method takes one parameters (used for office365):

* `$accesskey`: An accesstoken you get from https://login.microsoftonline.com/APP-ID/oauth2/v2.0/token (Read the next paragraph for more information)


The `authWithUserAndPass` method takes two parameters (used for on premises):

* `$username`: The user to connect to the server with. This is usually the
  local portion of the users email address. Example: "user" if the email address
  is "user@example.com".
* `$password`: The user's plain-text password.

Once you have your `\jamesiarmes\PhpEws\Client` object, you need to build your
request object. The type of object depends on the operation you are calling. If
you are using an IDE with code completion it should be able to help you
determine the correct classes to use using the provided docblocks.

The request objects are build similar to the XML body of the request. See the
resources section below for more information on building the requests.

## Prepare your Office 365 app for app-only authentication

As described in the [official microsoft documentation][9] you need to create a new application with a special authorization role.
After you have created the new application locate the requiredResourceAccess property in the manifest, and add the following inside the square brackets ([]):

```json
{
  "resourceAppId": "00000002-0000-0ff1-ce00-000000000000",
  "resourceAccess": [
    {
      "id": "dc890d15-9560-4a4c-9b7f-a736ec74ec40",
      "type": "Role"
    },
    {
      "id": "3b5f3d61-589b-4a3c-a359-5dd4b5ee5bd5",
      "type": "Scope"
    }
  ]
}
```

Return to the authorization page and select "Grant admin consent" for "Office 365 Exchange Online -> full_access_as_app".

 * Select Certificates & Secrets in the left-hand navigation under Manage
 * Select New client secret, enter a short description and select Add
 * Copy the Value of the newly added client secret and save it, you will need it later

Ensure that you are able to retrive the token for your app:

```php
$token = Cache::remember('ews_token', 3000, function () {
    $postOptions = [
        'http_errors' => false,
        'form_params' => [
            'client_id' => 'YOUR CLIENT ID',
            'client_secret' => 'YOUR SECRET',
            'grant_type' => 'client_credentials',
            'scope' => 'https://outlook.office365.com/.default'
        ]
    ];

    $url = 'https://login.microsoftonline.com/YOUR-TENANT/oauth2/v2.0/token'; //YOUR- TENANT is your primary domain (eg contoso.com)

    $client = new \GuzzleHttp\Client();
    $response = $client->request('POST', $url, $postOptions);
    $response = $response->getBody()->__toString();

    $response = json_decode($response);

    return $response->access_token;
});
```

Now you need to impersonate as a valid user mailbox:

```php
$client = new Client($this->host, $this->version);
$sid = new ConnectingSIDType();
$sid->SmtpAddress = "user@contoso.com";
$imp = new ExchangeImpersonationType();
$imp->ConnectingSID = $sid;
$client->setImpersonation($imp);
$client->authWithOauth2($token);
```

## Examples

There are a number of examples included in the examples directory. These
examples are meant to be run from the command line. In each, you will need to
set the connection information variables to match those of your Exchange server.
For some of them, you will also need to set ids or additional data that will be
used in the request.

## Resources

* [php-ews Website][4]
* [Exchange 2007 Web Services Reference][5]
* [Exchange 2010 Web Services Reference][4]
* [Exchange 2013 Web Services Reference][7]

## Support

All questions should use the [issue queue][8]. This allows the community to
contribute to and benefit from questions or issues you may have. Any support
requests received via email will be directed here.

[1]: https://scrutinizer-ci.com/g/jamesiarmes/php-ews
[2]: https://packagist.org/packages/php-ews/php-ews
[3]: https://github.com/jamesiarmes/php-ews/tree/master/examples/autodiscover
[4]: http://www.jamesarmes.com/php-ews/
[5]: http://msdn.microsoft.com/library/bb204119\(v=EXCHG.80\).aspx
[6]: http://msdn.microsoft.com/library/bb204119\(v=exchg.140\).aspx
[7]: http://msdn.microsoft.com/library/bb204119\(v=exchg.150\).aspx
[8]: https://github.com/jamesiarmes/php-ews/issues
[9]: https://learn.microsoft.com/en-us/exchange/client-developer/exchange-web-services/how-to-authenticate-an-ews-application-by-using-oauth
