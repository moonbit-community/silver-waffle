# Create a Code Review Bot

This part is divided as follows:

- Get the pull request's information
- Get the code comment from LLM
- Update the code comment from LLM

This part mainly serves as a referential implementation. Try to implement on your own before checking the following part.

## Get the Pull Request's Information

From the request body, we've already received a lot of information. We've extracted the installation id, but we still need more information. For example, we may want to know what has changed in this Pull Request:

### Task

Define another function to get the pull request's change from GitHub:

```moonbit
async fn get_pr_diff(
  owner : String,
  repo : String,
  pull_number : Int64,
  token : String,
  username : String
) -> String! {
  let request = @http.request!(
    "api.github.com",
    path="/repos/\{owner}/\{repo}/pulls/\{pull_number}",
    headers=@http.headers({
      "Accept": ["application/vnd.github.v3.diff"],
      "Authorization": [@encoding.encode(UTF8, "token \{token}")],
      "User-Agent": [@encoding.encode(UTF8, username)],
    }),
  )
  let response = @http.fetch!(request)
  if response.status() != 200 {
    let body = response.consume().unwrap()
    let content = @http.text(body).0.await!()
    response.drop()
    raise fail!("Error fetching diff \{response.status()}: \{content}")
  }
  let body = response.consume().unwrap()
  let content = @http.text(body).0.await!()
  response.drop()
  content
}
```

In the `top` function, we retrieve more information from the payload to get, for example, the pull request's number, owner, etc.:

```moonbit
async fn top(
  request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  // Return response to GitHub acknowledging receipt of the webhook
  let response = @http.response!(200)
  response_out.set(Ok(response))
  // Verify the header
  let headers = request.headers()
  guard headers.get("x-github-event") is [event] else {
    fail!("Expected x-github-event header")
  }
  guard @encoding.decoder(UTF8).decode!(Bytes::from_fixedarray(event))
    is "pull_request" else {
    fail!("Expected x-github-event to be pull_request")
  }
  // Parse the body
  let body = request.consume().unwrap()
  request.drop()
  let json = @http.json(body).0.await!()
  guard json
    is {
      "action": "opened"
      | "synchronize",
      "repository": {
        "name": String(repository),
        "owner": { "login": String(owner), .. },
        ..
      },
      "installation": { "id": Number(installation), .. },
      "pull_request": {
        "number": Number(pull_request),
        "body"? : pull_request_message,
        ..
      },
      ..
    }
  // Get the Pull Request message
  let pull_request_message = if pull_request_message is Some(String(message)) {
    message
  } else {
    ""
  }
  let pull_request_diff = 
  let environment = Map::from_array(@environment.get_environment())
  guard environment["CLIENT_ID"] is Some(client_id)
  guard environment["PRIVATE_KEY"] is Some(private_key)
  guard environment["USER_NAME"] is Some(username)
  let now = @wallClock.now().seconds
  let token = create_installation_access_token!(
    installation.to_uint(),
    private_key,
    client_id,
    now,
    username,
  )
  // Get the Pull Request's changes
  let pull_request_diff = get_pr_diff!(
    owner,
    repository,
    pull_request.to_int64(),
    token,
    username,
  )
  @io.println!(
    #|=== Pull Request Message ==
    $|\{pull_request_message}
    #|=== Pull Request Changes (in diff format) ==
    $|\{pull_request_diff}
  )
}
```

## Get the Comment

Now that we have enough information, we want to get code review from the LLM.

We've covered how to get information from LLM before. Now we only need to do it again with a different prompt.

### Task

Define another function to get code review from LLM:

```moonbit
async fn get_review(message : String, changes : String) -> String! {
  guard Map::from_array(@environment.get_environment()).get("OPENAI_API_KEY")
    is Some(token) else {
    fail!("OPENAI_API_KEY is not set")
  }
  let system_msg =
    #|You are a programmer conducting code review.
    #|Based on the pull request message and changes, provide feedback.
    #|
    #|Point out at most three problems that you observe
  let user_msg =
    #|=== Pull Request Message ==
    $|\{message}
    #|=== Pull Request Changes (in diff format) ==
    $|\{changes}
  let payload : Json = {
    "model": "deepseek/deepseek-chat",
    "messages": [
      { "role": "system", "content": String(system_msg) },
      { "role": "user", "content": String(user_msg) },
    ],
  }
  let request = @http.request!(
    "openrouter.ai",
    path="/api/v1/chat/completions",
    scheme=Https,
    method_=Post,
    headers=@http.headers({
      "Content-Type": ["application/json"],
      "Authorization": [@encoding.encode(UTF8, "Bearer \{token}")],
    }),
  )
  let body = request.body().unwrap()
  let output_stream = body.write().unwrap()
  @io.println!(payload.stringify(), stream=output_stream)
  output_stream.drop()
  body.finish(None).unwrap_or_error!()
  let response = @http.fetch!(request)
  let body = response.consume().unwrap()
  let content = @http.json(body).0.await!()
  response.drop()
  guard content
    is {
      "choices": [{ "message": { "content": String(content), .. }, .. }],
      ..
    } else {
    fail!("Unexpected response \{content}")
  }
  content
}
```

And apply it in `top`:

```moonbit
async fn top(
  request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  ...
  // Get the Pull Request's changes
  let pull_request_diff = get_pr_diff!(
    owner,
    repository,
    pull_request.to_int64(),
    token,
    username,
  )
  let review = get_review!(pull_request_message, pull_request_diff)
  @io.println!(
    #|=== Review ===
    $|\{review}
    ,
  )
}
```

## Update the Comment

Now that we have everything we need, we need to post the comment to the GitHub.

We need to first check if there's already an existing comment. If there is one, then we update it. Otherwise, we create one.

### Task

First, we define three functions for three GitHub API: list comments, create comment and update comment.

```moonbit
///|
async fn get_comment_list(
  owner : String,
  repo : String,
  issue_number : Int64,
  token : String,
  app_id : String,
  username : String
) -> Int64?! {
  let request = @http.request?(
    "api.github.com",
    path="/repos/\{owner}/\{repo}/issues/\{issue_number}/comments",
    headers=@http.headers({
      "Accept": ["application/vnd.github.v3+json"],
      "Authorization": [@encoding.encode(UTF8, "token \{token}")],
      "User-Agent": [@encoding.encode(UTF8, username)],
    }),
  ).unwrap()
  let response = @http.fetch!(request)
  if response.status() != 200 {
    let body = response.consume().unwrap()
    let content = @http.text(body).0.await!()
    response.drop()
    fail!("Failed to list issue comments. Reason: \{content}")
  }
  let body = response.consume().unwrap()
  let json = @http.json(body).0.await!()
  response.drop()
  guard json is Array(comments) else {
    fail!("Failed to list issue comments. Unexpected response: \{json}")
  }
  for comment in comments {
    if comment
      is {
        "performed_via_github_app": { "id": Number(github_app_id), .. },
        "id": Number(comment_id),
        ..
      } &&
      github_app_id.to_int64().to_string() == app_id {
      return Some(comment_id.to_int64())
    }
  } else {
    return None
  }
}

///|
async fn create_comment(
  owner : String,
  repo : String,
  issue_number : Int64,
  comment : String,
  token : String,
  username : String
) -> Unit! {
  let request = @http.request?(
    "api.github.com",
    path="/repos/\{owner}/\{repo}/issues/\{issue_number}/comments",
    method_=Post,
    headers=@http.headers({
      "Accept": ["application/vnd.github.v3+json"],
      "Authorization": [@encoding.encode(UTF8, "token \{token}")],
      "X-Github-Api-Version": ["2022-11-28"],
      "User-Agent": [@encoding.encode(UTF8, username)],
    }),
  ).unwrap()
  let body = request.body().unwrap()
  let output = body.write().unwrap()
  @io.println!(({ "body": String(comment) } : Json).stringify(), stream=output)
  output.drop()
  body.finish(None).unwrap()
  let response = @http.fetch!(request)
  if response.status() != 201 {
    let body = response.consume().unwrap()
    let content = @http.text(body).0.await!()
    response.drop()
    fail!("Failed to create comment. Reason: \{content}")
  }
  response.drop()
}

///|
async fn update_comment(
  owner : String,
  repo : String,
  comment_id : Int64,
  comment : String,
  token : String,
  username : String
) -> Unit! {
  let request = @http.request?(
    "api.github.com",
    path="/repos/\{owner}/\{repo}/issues/comments/\{comment_id}",
    method_=Patch,
    headers=@http.headers({
      "Accept": ["application/vnd.github.v3+json"],
      "Authorization": [@encoding.encode(UTF8, "token \{token}")],
      "X-Github-Api-Version": ["2022-11-28"],
      "User-Agent": [@encoding.encode(UTF8, username)],
    }),
  ).unwrap()
  let body = request.body().unwrap()
  let output = body.write().unwrap()
  @io.println!(({ "body": String(comment) } : Json).stringify(), stream=output)
  output.drop()
  body.finish(None).unwrap()
  let response = @http.fetch!(request)
  if response.status() != 200 {
    let body = response.consume().unwrap()
    let content = @http.text(body).0.await!()
    response.drop()
    fail!("Failed to update comment. Reason: \{content}")
  }
  response.drop()
}
```

Then at the end of the `top`, we use these functions. Notice that we also need to add an environment variable of APP ID, which you can also retrieve from the GitHub APP setting page.

```moonbit
async fn top(
  request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  ...
  guard environment["APP_ID"] is Some(app_id)
  ...
  let review = get_review!(pull_request_message, pull_request_diff)
  let comment_id = get_comment_list!(
    owner,
    repository,
    pull_request.to_int64(),
    token,
    app_id,
    username,
  )
  if comment_id is Some(id) {
    update_comment!(owner, repository, id, review, token, username)
  } else {
    create_comment!(
      owner,
      repository,
      pull_request.to_int64(),
      review,
      token,
      username,
    )
  }
}
```

## Bonus: Execute in Parallel

If we observe what we've written, even though we are using `async` everywhere, what we have is actually synchronous: we wait for the result of the previous operation and then we perform the next operation.

However, we can execute things that are not dependent on each other in parallel. For example, we can start sending the HTTP request and then write into the body; we can also check if the comment exists while performing code review by LLM.

### Task

Take the example of main logic, here's what we can do:

```moonbit
async fn top(
  request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  ...
  let token = create_installation_access_token!(
    installation.to_uint(),
    private_key,
    client_id,
    now,
    username,
  )
  let review = @promise.spawn(async fn(_defer) {
    let pull_request_diff = get_pr_diff!(
      owner,
      repository,
      pull_request.to_int64(),
      token,
      username,
    )
    get_review!(pull_request_message, pull_request_diff)
  })
  let comment_id = get_comment_list!(
    owner,
    repository,
    pull_request.to_int64(),
    token,
    app_id,
    username,
  )
  if comment_id is Some(id) {
    update_comment!(owner, repository, id, review.await!(), token, username)
  } else {
    create_comment!(
      owner,
      repository,
      pull_request.to_int64(),
      review.await!(),
      token,
      username,
    )
  }
}
```

And for sending requests, here's what we can do:

```moonbit
async fn get_review(message : String, changes : String) -> String! {
  ...
  let request = @http.request!(
    "openrouter.ai",
    path="/api/v1/chat/completions",
    scheme=Https,
    method_=Post,
    headers=@http.headers({
      "Content-Type": ["application/json"],
      "Authorization": [@encoding.encode(UTF8, "Bearer \{token}")],
    }),
  )
  let body = request.body().unwrap()
  let output_stream = body.write().unwrap()
  @promise.spawn(async fn(defer) {
    defer(fn() {
      output_stream.drop()
      body.finish(None).unwrap()
    })
    @io.println!(payload.stringify(), stream=output_stream)
  })
  |> ignore
  let response = @http.fetch!(request)
  let body = response.consume().unwrap()
  let content = @http.json(body).0.await!()
  response.drop()
  ...
}
```

## What's next

I hope you have enjoyed this journey. But this is just the beginning.

From the functionality perspective, you may provide all the functionalities as tools (<https://openrouter.ai/docs/features/tool-calling>) to the LLM and let LLM choose what to use. From the production perspective, you may want to integrate with observability stack such as LGTM stack. 

Possibilities are limitless, so

¡Buena suerte!
