# Create a GitHub App

This part is divided as follows:

- Register on GitHub and create an App
- Create a server that handles the webhook
- Generate an access token based on the request

## Create a GitHub App

In this section, we define a GitHub App and get the necessary information for the next steps.

```{note}
Some instructions are excerpts from <https://docs.github.com>.
```

### Task

- Prepare a callback address. Visit <https://smee.io> and click **Start a new channel**. Keep the **Webhook Proxy URL** for the following parts.  
  It's fine if you've lost it as you may start new channels later, but be sure to update where it's used.
- Register on GitHub if you don't have an account yet. Visit <https://github.com> and sign up. Follow the prompts to create your account.
  ```{seealso}
  <https://docs.github.com/en/get-started/start-your-journey/creating-an-account-on-github>
  ```
- Create a GitHub App. 
  1. Navigate to your account setting and choose **Developer settings** on the left sidebar. 
  2. Click **GitHub Apps** on the left sidebar and click **New GitHub App**.
  3. Fill out the necessary information. You may come back and adjust them later.  
     For **Webhook URL**, put the Webhook Proxy URL obtained perviously.  
     For permissions, we need **Pull requests** (read and write) and **Contents** (read) in **Repository permission**.  
     For events, we subscribe to **Pull request**.
  4. After creation, we go to the application setting page and get the **Client ID**, and the private key by going to `Private keys` section and generate one. The `pem` file will be downloaded.
  ```{seealso}
  <https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app>
  ```

### Explanation

We created a GitHub App that subscribes to pull requests events, which will be sent through webhook. The webhook will be delivered through smee to your locally running application. You can later change it to your properly hosted application. 

As a code review agent, we expect to read the content of the pull request and the content in the repository, and then write comments to the pull request, thus permission is required.

The private key is needed to sign the access token so that GitHub would know that it's us sending the requests.

## Create a Server that handles the webhook

In this section, we create a server that handles the webhook. We will start the proxy server and then proxy to our simple server.

Apart from that, we also need a repository to show that it does work.

### Task

On one side:
1. Create a server as in previous task [Create a Server](#create-a-server). Print to `@io.stdout` the request headers and the request body as `Json`, and return a response with 200 and `Hello World` in the body by rewriting `src/stub.mbt` to:
   ```moonbit
   async fn top(
     request : @types.IncomingRequest,
     response_out : @types.ResponseOutparam
   ) -> Unit! {
     let headers = request.headers()
     let entries = headers.entries()
     let body = request.consume().unwrap()
     headers.drop()
     request.drop()
     let json = @http.json(body).0.await!!()
     @io.println("Request Headers:").await!!()
     for entry in entries {
       @io.print("\n\{entry.0}: ").await!!()
       @io.write(entry.1, @io.stdout).await!!()
     }
     @io.println("\nContent: \{json.stringify()}").await!!()
     let response = @types.OutgoingResponse::outgoing_response(@http.headers({}))
     response.set_status_code(200).unwrap()
     response_out.set(Ok(response))
   }
   ```
2. In a terminal, execute `smee -u {your-web-hook-url} -p 8080`, where your web hook url is the one that smee assigned to you in the previous step. You should see:  
   ```
   Forwarding https://smee.io/xxx to http://127.0.0.1:8080/
   Connected https://smee.io/xxx
   ```

On the other side:
1. Prepare a repository. You can use this template <https://github.com/peter-jerry-ye/http-template> and create a new repository.
2. Install your application on the repository. Go to the setting page of your GitHub App and choose **Install App** on the left sidebar. Install for yourself and choose at least the repository that you've just created.
3. Create a pull request in the repository. You can edit `README.md` on the website, make some changes, and commit changes by creating a **new branch** for the commit and start a pull request.

When you create a pull request, you should observe the output of both the `smee`:
```
POST http://127.0.0.1:8080 - 200
```

and your server which consists of pairs of headers and a super larger JSON.

### Explanation

We created a server that prints the details of the payloads for us. You can also observe the payload on GitHub by choosing **Advanced** panel on the setting page of your GitHub App.

We've subscribed to pull request events, so when we create, update or edit a pull request, we will receive such events. In the example where we've created a pull request, the `x-github-event` header of the request is `pull_request`, and the `action` field in the JSON body is `opened`. When the user push something, it will be `synchronize`.

```{seealso}
For full information on pull request payload, checkout GitHub document <https://docs.github.com/en/webhooks/webhook-events-and-payloads#pull_request>.
```

## Create an Access Token

When we access APIs of GitHub, we need to have an access toekn to make requests on behalf of the GitHub App so that:

- Have higher rate limit (at least 5000/h per installation vs. 60/h per IP address if not authenticated).
- Have access to private repository.

### Task

Add a function in `src/stub.mbt` and adjust the `top`:

```moonbit
async fn create_installation_access_token(
  installation : UInt,
  private_key : String,
  client_id : String,
  now : UInt64,
  username : String
) -> String! {
  let jwt = @jwt.sign_rs256(
    {
      "iat": Number((now - 60).to_double()),
      "exp": Number((now + 600).to_double()),
      "iss": String(client_id),
    },
    private_key,
  )
  let request = @http.request!(
    "api.github.com",
    method_=Post,
    path="/app/installations/\{installation}/access_tokens",
    headers=@http.headers({
      "Accept": [b"application/vnd.github.v3+json".to_fixedarray()],
      "X-GitHub-Api-Version": [b"2022-11-28".to_fixedarray()],
      "User-Agent": [@encoding.encode(UTF8, username).to_fixedarray()],
      "Authorization": [@encoding.encode(UTF8, "Bearer \{jwt}").to_fixedarray()],
    }),
  )
  let response = @http.fetch(request).await!!()
  if response.status() != 201 {
    let body = response.consume().unwrap()
    let content = @http.text(body).0.await!!()
    fail!("Failed to create installation access token. Reason: \{content}")
  }
  let body = response.consume().unwrap()
  let json = @http.json(body).0.await!!()
  guard json is { "token": String(token), .. }
  token
}

async fn top(
  request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  ...
  let json = @http.json(body).0.await!!()
  guard json is { "installation": { "id": Number(installation), .. }, .. }
  let environment = Map::from_array(@environment.get_environment())
  guard environment["CLIENT_ID"] is Some(client_id)
  guard environment["PRIVATE_KEY"] is Some(private_key)
  guard environment["USER_NAME"] is Some(username)
  let now = @wallClock.now().seconds
  let token = create_installation_access_token!!(
    installation.to_uint(),
    private_key,
    client_id,
    now,
    username,
  )
  @io.println("Got token \{token}").await!!()
  let response = @types.OutgoingResponse::outgoing_response(@http.headers({}))
  ...
}
```

Add dependency:

```bash
moon add peter-jerry-ye/utils
```

Update `src/moon.pkg.json`:

```json
{
  "link": {
    // ...
  },
  "import": [
    // ...
    "peter-jerry-ye/utils/jwt",
    "moonbitlang/x/encoding",
    "peter-jerry-ye/wasi-imports/interface/wasi/clocks/wallClock",
    "peter-jerry-ye/wasi-imports/interface/wasi/cli/environment"
  ]
}
```

Rewrite `wit/imports.wit` with

```wit
package moonbit-community:server;

world proxy {
  include wasi:http/proxy@0.2.0;
  import wasi:cli/environment@0.2.0;
}
```

Serve your application:

```bash
USER_NAME=your-user-name CLIENT_ID=your-client-id PRIVATE_KEY=your-private-key wasmtime serve -S cli=y --env USER_NAME --env CLIENT_ID --env PRIVATE_KEY target/server.wasm
```

```{hint}
You may use tools such as dotenvx (<https://dotenvx.com/>) for managing the environment variables and autoloading during execution.
```

### Explanation

We retrieved the installation from the request header, the client id and private key from the environment, and the time using `wasi-clock`.
Then we created a JWT using predefined helper function. We send this JWT to the endpoint of GitHub to retrieve an access token. This will be used later for accessing the resources of the installation.

The GitHub API request users to send with a `user-agent` and

> GitHub recommends using your GitHub username, or the name of your application, for the User-Agent header value. This allows GitHub to contact you if there are problems.

Thus we retrieve them from the environment.

```{seealso}
For sending requests to GitHub, checkout GitHub documentation <https://docs.github.com/en/rest/using-the-rest-api/getting-started-with-the-rest-api?apiVersion=2022-11-28>

For full information on access token, checkout GitHub documentation <https://docs.github.com/en/rest/apps/apps?apiVersion=2022-11-28#create-an-installation-access-token-for-an-app>
```

### Exercise

Try to implement the third party by yourself.