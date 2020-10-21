<div align="center">
<h1>🔐🙅‍♀️ workers-auth0-example</h1>
<p>authorization/authentication at the edge, using <a href="https://workers.dev">cloudflare workers</a> and <a href="https://auth0.com">auth0</a></p>
</div>

## tutorial

to learn about how to build this project and explore use-cases, [check out the tutorial!](https://developers.cloudflare.com/workers/tutorials/authorize-users-with-auth0)

## prerequisites

- a cloudflare workers unlimited account
- an auth0 account and a configured application
- if deploying in front of a domain, a configured cloudflare zone (see the "origin" section of "deploys" below)
- [`wrangler`][wrangler] cli tool installed and configured (see the [quick start][quick start] in the docs)

## getting started

using `wrangler`, you can generate a new project using this repo:

`wrangler generate my-auth-example https://github.com/signalnerve/workers-auth0-example`

## setup

this project makes heavy use of [`wrangler`][wrangler], the workers command-line tool, for managing environment variables and deploying this codebase. before trying to deploy this project, i recommend closely following this section of the readme to get all the relevant tokens, ids, and values that you'll need to successfully deploy this project.

### `wrangler.toml`

the `wrangler.toml` config file in this repository needs to be configured for deploying your own version of this application. using `wrangler generate` will automatically add a `name` field - you'll also need to configure an account id, and, depending on if you're deploying your application to a _zone_ (see "deploying" later in this readme), a zone id and route.

for more information on configuring this correctly, i strongly recommend you check out the [quick start][quick start] in the workers docs!

_this section is still in progress!_

#### kv namespace creation

by default, this project uses the binding `AUTH_STORE` to refer to a kv namespace where credentials are read/written from the workers script. to create your own kv namespace, use `wrangler`, and copy the resulting id into the `kv-namespaces` portion of `wrangler.toml`:

```
wrangler kv:namespace create AUTH_STORE
```

### secrets

below is a table of secrets that the workers script will look for when it processes a client request. each should be set with `wrangler secret`:

| wrangler secret key | value                                                                            |
| ------------------- | -------------------------------------------------------------------------------- |
| AUTH0_DOMAIN        | your auth0 domain (e.g. `myapp.auth0.com`)                                       |
| AUTH0_CLIENT_ID     | your auth0 client id                                                             |
| AUTH0_CLIENT_SECRET | your auth0 client secret                                                         |
| AUTH0_CALLBACK_URL  | the callback url for your application (see below)                                |
| SALT                | A secret string used to encrypt user `sub` values (see "Setting the salt" below) |

### setting the callback url

in order to correctly set the callback url for your application, you will need to determine where your application will be deployed. regardless of whether you're setting up a _originless_ or _origin_-based deploy, the callback handler for this project is defined at `/auth`. this means that if you're testing or deploying a staging version of this project, your callback url will likely be something like `my-auth-example.signalnerve.workers.dev/auth`, or for production, you should set it to something like `my-production-app.com/auth`.

### setting the salt

in order to safely store user IDs (the `sub` value from Auth0), we should always refer to them by an encrypted value, which we can generate using the `crypto.subtle.digest` function in the web crypto api. in order to do this, we need to set a _salt_: a secret value that is included in the text we're encrypting.

cloudflare provides an api for random data at `csprng.xyz`: visit `https://csprng.xyz/v1/api`, and copy the `Data` field to your clipboard. if you'd like to generate a string yourself, remember that it's important that the salt can't easily be guessed!

with a random string generated, you can set it using `wrangler secret`:

```sh
$ wrangler secret put SALT
```

### allowed origin/callback urls

note that auth0 has great security defaults, and any callback urls or origins that you attempt to login from need to be explicitly provided in the auth0 dashboard as part of your application config. using the above `workers.dev` example, you should ensure the following values are set in your application's settings, along with any additional urls used as part of testing (e.g. `localhost:8787` for [wrangler dev][wrangler dev] usage):

| allowed origins                         | allowed callback urls                        |
| --------------------------------------- | -------------------------------------------- |
| my-auth-example.signalnerve.workers.dev | my-auth-example.signalnerve.workers.dev/auth |

## configuration

this application features a growing number of configuration options available via the `config` object in `index.js`. these can be customized to toggle different features in the script, and are outlined below (with callouts to more detailed explanations, further in the readme):

### `config.hydrateState`

enable authorization state hydration, to pass user info such as name and email to your application. see "edge state hydration" later in the readme.

### `config.originless`

serve responses directly from the edge, without the need for an origin server. see "deploying" later in the readme.

## edge state hydration

by default, any information stored in auth0, such as your user's name, email, or other custom values (see ["Define and Maintain Custom User Data" from auth0's docs][auth0 custom data]) can be inlined into your website's response by defining `script#edge_state` tag in the `head` section of your static site. workers' [htmlrewriter][htmlrewriter] will inline the provided state into that tag as a JSON string. you can opt into this feature by pasting something similar to the below snippet:

```html
<script id="edge_state" type="application/json">
  {}
</script>
```

note that if you'd like to disable this feature, you can toggle the config option `hydrateState` to `false`.

## development

local dev can be very closely simulated using [wrangler dev][wrangler dev]. note that `wrangler`/workers does not support specific secrets for `wrangler dev`. you should set up a different environment such as `development`, and use it in your local testing, in order to allow auth0 to successfully redirect you to `wrangler dev`'s `localhost:8787` set up (or use something like [`ngrok`][ngrok] for non-localhost usage). sane default environments for this repo are in development, see [this issue](https://github.com/signalnerve/workers-auth0-example/issues/4) for more details.

## deploying

`wrangler publish`

### originless

by default, this codebase is deployed without an origin - we call this "originless" - and can be used to serve responses "from the edge". this means that your application can be authenticated/authorized at a cloudflare edge server close to your users, which can make a huge impact on latency.

this project uses [workers sites][workers sites] to serve static websites from the edge. by default, any assets in the `public` folder will be uploaded to workers kv, and used to serve your static site to users. if you'd like to change this from `public`, modify the `site.bucket` field in `wrangler.toml` to point to your application. if you're using `gatsby` or similar static site frameworks, this should represent the _final, compiled_ version of your site.

### origin

if you'd like to authorize/authenticate users at the edge, but still return a response from an origin server, set `config.originless` to `false`. this requires that your workers script is deployed to a zone (see ["configure" and "publish to your domain" in the cloudflare workers quick start][quick start]). _this hasn't been tested and may be broken. sorry, coming soon!_

## issues

please file issues for bugs and feature requests on this project! note that this repo isn't a good place to get help with wrangler, cloudflare workers, or other specific platform issues: check out the [workers forums](https://community.cloudflare.com/c/developers/workers/40).

## license

mit

[auth0 custom data]: https://auth0.com/docs/microsites/manage-users/define-maintain-custom-user-data
[htmlrewriter]: https://developers.cloudflare.com/workers/reference/apis/html-rewriter/
[ngrok]: https://ngrok.com/
[quick start]: https://developers.cloudflare.com/workers/quickstart#configure
[workers sites]: https://developers.cloudflare.com/workers/sites
[wrangler]: https://github.com/cloudflare/wrangler
[wrangler dev]: https://github.com/cloudflare/wrangler#-dev
