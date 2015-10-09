## Passwordless authentication

**This feature is experimental**

See also [UI passwordless](../ui/ui-passwordless.md)

Local passwordless or email only authentication can be configured easily by adding a passwordless
object to the providers section of your configuration files.

```json
{
  // ...
  "providers": {
    "passwordless": {
      "tokenTTL": 600
    }
  }
}
```

The tokenTTL setting specifies the number of seconds before the token associated with the email link send to the user expires. The default is 900 seconds or 15 minutes.

The passwordless provider also requires that the mailer is configured ([mailer.md](../server/mailer.md))
