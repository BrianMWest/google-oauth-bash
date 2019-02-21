# google-oauth-bash

## Introduction and my pain

A tool for getting an Oauth2 token for a Google API Service Account using Bash.
I was creating a CI/CD script for a Rust project with GitLab CI and wanted the
finished result to be published to a URL that could be shared with others.
The idea was that the script would upload to a file server and then send a link
to the new version in a Discord channel.

You will need `jq` and `openssl` for this to work.

I chose Google Cloud Storage due to perceived simplicity of use and retrieval
as well as the low cost for the usage that I expected to see.

Discord was easy because you can create a Webhook URL and simply POST a JSON
payload to it with cURL.
The Google JSON API was moderately more difficult for a couple of reasons:

1. I could not mount files in.
2. I was running a minimal Docker image `rust:latest` rather than creating a custom
   image with the Google client libraries installed.
3. This was a service account (server to server) job that ran without user interactions.

The GitLab CI allows setting environment variables, but it doesn't let you mount files.
To get this to work, I created a service account in Google's IAM and then downloaded a JSON
file containing the authentication details. You need two things from here.
First, you need the `private_key`, which you can extract with `jq -r '.private_key' < credentials.json`.
Second, you need the `client_email`, which can be extracted similarly.

I set these both as environment variables in my GitLab CI Pipeline:
`GCP_CLIENT_EMAIL` and `GCP_PRIVATE_KEY`.

Finally, you will need a `GCP_SCOPE` URL. The particular API that you are using
will tell you what the scope URL options are.

Once these are set, this script will hit the correct Google Oauth2 token
generation endpoint for a service account and return the authorization token
that you need for future requests. The token is good for an hour from what I saw,
although your mileage may vary.

Once you have the authorization token, you can use the API by providing your Oauth2
token through a header `Authorization: Bearer ${OAUTH_TOKEN}`. Technically,
the "Bearer" is defined by the Oauth2 response under `.token_type`, but it is
always "Bearer."

Much of this is documented in Google's references, although it is significantly
caveated for Bash/cURL as "Don't do this because you'll just mess it up anyway."
You can find it here:
[Google Identity Platform - Using OAuth 2.0 for Server to Server Applications](https://developers.google.com/identity/protocols/OAuth2ServiceAccount).

I encourage you to take this script and change it do whatever you need. This worked
for my use case, but it might not work for others. But I felt that this would
be a better place to start for other people compared to where I had to start to
figure this out. Good luck.

## Usage

Here is an example of uploading `my_file.exe` to `<bucket>` as `<key-and-name>` (note
that the full destination path needs to be provided under `name=`, and it should
be url-encoded):

```
OAUTH_TOKEN=$(google-auth -k "$GCP_PRIVATE_KEY" -s "$GCP_SCOPE" -e "$GCP_CLIENT_EMAIL")

declare -r API="https://www.googleapis.com/upload/storage/v1"
declare -r UPLOAD_URL=$(curl -s -X POST \
                             --data-binary @./my_file.exe \
                             -H "Authorization: Bearer ${OAUTH_TOKEN}" \
                             "${API}/b/<bucket>/o/?name=<key-and-name>&uploadType=media" | jq -r '.mediaLink')
```
