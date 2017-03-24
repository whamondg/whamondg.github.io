---
layout: post
title: Building a GitHub event listener with Elixir and Phoenix (part 1 - the basics)
---

# Introduction

Using [GitHub's webhooks API](https://developer.github.com/webhooks/) it's simple to get notifications about all kinds of repository activity.  These events include commits, pull requests, and wiki updates.  It can also be used to orchestrate software deployment.

Building a tool to respond to these events requires listening for HTTP requests, processing data sent in the request payload, and the orchestration of downstream services.  It's a nice problem space for playing around with web frameworks and across a series of blog posts I'm going to use it to explore [Elixir](http://elixir-lang.org/) and [Phoenix](http://www.phoenixframework.org/).

Elixir is a functional language which compiles into BEAM byte code that can be run on the [Erlang](https://www.erlang.org/) VM.  Leveraging Erlang gives Elixir a mature runtime environment with built in support for distibution, concurrency, and fault tolerance.  Phoenix is a modular web framework written in Elixir which has been designed to be easy to use, reliable, and highly performant.

The ultimate aim is to create an application that listens for HTTP requests triggered by GitHub webhooks.  Upon receiving a valid request subsequent actions will be triggered such as logging the events, calling a remote webservice, etc.


## What's in a name?

I'm calling the application "githooker" after the Rugby Union position in the front row of the scrum.  The hooker's job is to win the ball by hooking it with their feet and pass it back for other members of the team to take ownership.  It's a nice analogy with our application passing on GitHub events to other services.


# Prerequisites

Before getting started we need to install a few things:

  * Firstly, [Elixir and its dependencies](http://elixir-lang.org/install.html)  
  * Next, [Phoenix](http://www.phoenixframework.org/docs/installation)
  * Finally, to use Phoenix's default database support we also need to install [PostgreSQL](https://www.postgresql.org/download/).


# Getting started

## Creating the application

Phoenix provides a mechanism to quickly get started and create the basic file structure of a new application by running the following command: 


```shell
> mix phoenix.new githooker
```

The generated application includes support for web assets using [Brunch](http://brunch.io/) and database models using [Ecto](https://github.com/elixir-ecto/ecto), both of which can be easily disabled if necessary.

By default Ecto uses a PostgreSQL adapter, but this can be changed by editing the development and test database configuration files, found in `githooker/config/dev.exs` and `githooker/config/test.exs` respectively.  These files also contain the credentials used to connect to the database, which can be changed as required.  It's important to note that the role used to connect to PostgreSQL needs to have the CREATEDB permission granted.

Production configuration can be found in `githooker/config/prod.secret.exs`.


## Testing out of the box

Our new Phoenix application contains several tests already, which can be run with the following command:


```shell
> mix test
```

This will compile the code and run the tests, which will pass as long as the database configuration has been set up correctly.


## A good time to think about version control

It's worth getting all this into version control before we start breaking things.  Phoenix provides a .gitignore file with sensible defaults to stop things like node_modules being committed.  It's also worth noting that production config will not be accidentally commited since the file `githooker/config/prod.secret.exs` is ignored.


## Running the application

To start Githooker and take a look at our new Phoenix app run the following:


```shell
> cd githooker
> mix phoenix.server
```

This starts our server, which will listen to port 4000 by default.  Pointing the browser to [http://localhost:4000](http://localhost:4000) shows a "Welcome to Phoenix page", which includes links to the [Phoenix documentation](http://www.phoenixframework.org/docs) and other useful resources.

![Default Phoenix home page](/img/welcome_to_phoenix.png)

With those simple steps we have a new web application up and running and it's time to do something a bit more useful by adding out first feature.


# Creating a receiver for GitHub webhooks

## Writing the tests

We're going to build up functionality slowly, so initially GitHooker will respond to a single event type with HTTP 200 Success.  Every other type of event will receive HTTP 400 Bad request.

We start with the tests by creating `githooker/test/controllers/githook_controller_test.exs` 

```exs
defmodule Githooker.GithookControllerTest do
  use Githooker.ConnCase

  test "Github deploy events are handled", %{conn: conn} do
    test_event = "deployment"

    result = conn
    |> put_req_header("x-github-event", test_event)
    |> post("/api/githook")

    assert json_response(result, 200) == %{"status" => "OK", "event" => test_event}
  end

  test "Unknown Github events are not handled", %{conn: conn} do
    test_event = "some unhandled event"

    result = conn
    |> put_req_header("x-github-event", test_event)
    |> post("/api/githook")

    assert json_response(result, 400) == %{"status" => "Unknown event", "event" => test_event}
  end
end
```

This new file contains two simple enough tests that send HTTP requests to an api endpoint and assert an appropriate response is received, both of which will fail:


```shell
› mix test
...

  1) test Unknown Github events are not handled (Githooker.GithookControllerTest)
     test/controllers/githook_controller_test.exs:15
     ** (RuntimeError) expected response with status 400, got: 404
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:328: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:374: Phoenix.ConnTest.json_response/2
       test/controllers/githook_controller_test.exs:22



  2) test Github deploy events are handled (Githooker.GithookControllerTest)
     test/controllers/githook_controller_test.exs:5
     ** (RuntimeError) expected response with status 200, got: 404
     stacktrace:
       (phoenix) lib/phoenix/test/conn_test.ex:328: Phoenix.ConnTest.response/2
       (phoenix) lib/phoenix/test/conn_test.ex:374: Phoenix.ConnTest.json_response/2
       test/controllers/githook_controller_test.exs:12

.

Finished in 0.1 seconds 
6 tests, 2 failures

Randomized with seed 766866

```

The exact output might not be the same as the example shown since the tests are automatically run asynchronously.  However, even though the order of the output may vary, the failure reason should be the same.


## Making the tests pass

The tests are failing because Phoenix doesn't know anything about the `/api/githook` path that is being requested so it returns a HTTP 404 response.  The first step towards fixing this problem is to create a Phoenix route.


### Creating the route

We add a new route to Phoenix by editing: `githooker/web/router.ex`.  

By default, an api pipeline is configured but it's effectively disabled since it's commented out and not scoped to a particular URI path.  This can be remedied by uncommenting the scope section that maps `/api` to the api pipeline and adding a line of config to map POST requests that hit `/githook` to a controller.

The completed changes to the `githooker/web/router.ex` file look like this:


```exs
defmodule Githooker.Router do
  use Githooker.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/", Githooker do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
  end

  scope "/api", Githooker do
     pipe_through :api

     post "/githook", GithookController, :webhook
  end
end
```

Our application is now configured to handle POST requests to '/api/githook' by routing them to a controller called GithookController.  Since this controller doesn't exist we can expect our tests to fail with a new error:

```shell
› mix test
Generated githooker app
...

  1) test Github deploy events are handled (Githooker.GithookControllerTest)
     test/controllers/githook_controller_test.exs:5
     ** (UndefinedFunctionError) function Githooker.GithookController.init/1 is undefined (module Githooker.GithookController is not available)
     stacktrace:
       Githooker.GithookController.init(:webhook)
       (githooker) web/router.ex:1: anonymous fn/1 in Githooker.Router.match_route/4
       (githooker) lib/phoenix/router.ex:261: Githooker.Router.dispatch/2
       (githooker) web/router.ex:1: Githooker.Router.do_call/2
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.phoenix_pipeline/1
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/githook_controller_test.exs:10: (test)



  2) test Unknown Github events are not handled (Githooker.GithookControllerTest)
     test/controllers/githook_controller_test.exs:15
     ** (UndefinedFunctionError) function Githooker.GithookController.init/1 is undefined (module Githooker.GithookController is not available)
     stacktrace:
       Githooker.GithookController.init(:webhook)
       (githooker) web/router.ex:1: anonymous fn/1 in Githooker.Router.match_route/4
       (githooker) lib/phoenix/router.ex:261: Githooker.Router.dispatch/2
       (githooker) web/router.ex:1: Githooker.Router.do_call/2
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.phoenix_pipeline/1
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/githook_controller_test.exs:20: (test)

.

Finished in 0.1 seconds
6 tests, 2 failures

Randomized with seed 847045
```


### Adding the controller

As expected the tests are failing because the GithookController doesn't exist.  We can easily fix this error by adding the missing controller.  Create the file `web/controllers/githook_controller.ex` with the following content:

```exs
defmodule Githooker.GithookController do
  use Githooker.Web, :controller

end

```

At this point the controller exists but the "webhook" function configured to handle POST requests to "/githook" is still missing.  Running the tests confirms what we already know:

```shell
› mix test
Compiling 1 file (.ex)
...

  1) test Unknown Github events are not handled (Githooker.GithookControllerTest)
     test/controllers/githook_controller_test.exs:14
     ** (UndefinedFunctionError) function Githooker.GithookController.webhook/2 is undefined or private
     stacktrace:
       (githooker) Githooker.GithookController.webhook(%Plug.Conn{adapter: {Plug.Adapters.Test.Conn, :...}, assigns: %{}, before_send: [#Function<1.106838844/1 in Plug.Logger.call/2>], body_params: %{}, cookies: %Plug.Conn.Unfetched{aspect: :cookies}, halted: false, host: "www.example.com", method: "POST", owner: #PID<0.328.0>, params: %{}, path_info: ["api", "githook"], path_params: %{}, peer: {{127, 0, 0, 1}, 111317}, port: 80, private: %{Githooker.Router => {[], %{}}, :phoenix_action => :webhook, :phoenix_controller => Githooker.GithookController, :phoenix_endpoint => Githooker.Endpoint, :phoenix_format => "json", :phoenix_layout => {Githooker.LayoutView, :app}, :phoenix_pipelines => [:api], :phoenix_recycled => true, :phoenix_route => #Function<1.49509388/1 in Githooker.Router.match_route/4>, :phoenix_router => Githooker.Router, :phoenix_view => Githooker.GithookView, :plug_session_fetch => #Function<1.61377594/1 in Plug.Session.fetch_session/1>, :plug_skip_csrf_protection => true}, query_params: %{}, query_string: "", remote_ip: {127, 0, 0, 1}, req_cookies: %Plug.Conn.Unfetched{aspect: :cookies}, req_headers: [{"x-github-event", "some unhandled event"}], request_path: "/api/githook", resp_body: nil, resp_cookies: %{}, resp_headers: [{"cache-control", "max-age=0, private, must-revalidate"}, {"x-request-id", "the9b9524h0bo8cbplb5cdjp2ttd28g7"}], scheme: :http, script_name: [], secret_key_base: "UKwac8wGH295Mo0C1HqY28ESvgTHHCUF1uvytmFQMFzpCSg2wjX0xIuCEj5HhSnj", state: :unset, status: nil}, %{})
       (githooker) web/controllers/githook_controller.ex:1: Githooker.GithookController.action/2
       (githooker) web/controllers/githook_controller.ex:1: Githooker.GithookController.phoenix_controller_pipeline/2
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.instrument/4
       (githooker) lib/phoenix/router.ex:261: Githooker.Router.dispatch/2
       (githooker) web/router.ex:1: Githooker.Router.do_call/2
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.phoenix_pipeline/1
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/githook_controller_test.exs:19: (test)



  2) test Github deploy events are handled (Githooker.GithookControllerTest)
     test/controllers/githook_controller_test.exs:4
     ** (UndefinedFunctionError) function Githooker.GithookController.webhook/2 is undefined or private
     stacktrace:
       (githooker) Githooker.GithookController.webhook(%Plug.Conn{adapter: {Plug.Adapters.Test.Conn, :...}, assigns: %{}, before_send: [#Function<1.106838844/1 in Plug.Logger.call/2>], body_params: %{}, cookies: %Plug.Conn.Unfetched{aspect: :cookies}, halted: false, host: "www.example.com", method: "POST", owner: #PID<0.330.0>, params: %{}, path_info: ["api", "githook"], path_params: %{}, peer: {{127, 0, 0, 1}, 111317}, port: 80, private: %{Githooker.Router => {[], %{}}, :phoenix_action => :webhook, :phoenix_controller => Githooker.GithookController, :phoenix_endpoint => Githooker.Endpoint, :phoenix_format => "json", :phoenix_layout => {Githooker.LayoutView, :app}, :phoenix_pipelines => [:api], :phoenix_recycled => true, :phoenix_route => #Function<1.49509388/1 in Githooker.Router.match_route/4>, :phoenix_router => Githooker.Router, :phoenix_view => Githooker.GithookView, :plug_session_fetch => #Function<1.61377594/1 in Plug.Session.fetch_session/1>, :plug_skip_csrf_protection => true}, query_params: %{}, query_string: "", remote_ip: {127, 0, 0, 1}, req_cookies: %Plug.Conn.Unfetched{aspect: :cookies}, req_headers: [{"x-github-event", "deployment"}], request_path: "/api/githook", resp_body: nil, resp_cookies: %{}, resp_headers: [{"cache-control", "max-age=0, private, must-revalidate"}, {"x-request-id", "lh2icpittf7vfpddclp737shgihi5kc2"}], scheme: :http, script_name: [], secret_key_base: "UKwac8wGH295Mo0C1HqY28ESvgTHHCUF1uvytmFQMFzpCSg2wjX0xIuCEj5HhSnj", state: :unset, status: nil}, %{})
       (githooker) web/controllers/githook_controller.ex:1: Githooker.GithookController.action/2
       (githooker) web/controllers/githook_controller.ex:1: Githooker.GithookController.phoenix_controller_pipeline/2
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.instrument/4
       (githooker) lib/phoenix/router.ex:261: Githooker.Router.dispatch/2
       (githooker) web/router.ex:1: Githooker.Router.do_call/2
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.phoenix_pipeline/1
       (githooker) lib/githooker/endpoint.ex:1: Githooker.Endpoint.call/2
       (phoenix) lib/phoenix/test/conn_test.ex:224: Phoenix.ConnTest.dispatch/5
       test/controllers/githook_controller_test.exs:9: (test)

.

Finished in 0.1 seconds
6 tests, 2 failures

Randomized with seed 131604
```

The easiest way to get a new failure from the tests is to add an empty webhook function to our new controller.  This simple change looks as follows:


```exs
defmodule Githooker.GithookController do
  use Githooker.Web, :controller

  def webhook(conn, _params) do
  end
end
```

As expected we now get a different error from the tests:


```shell
 ** (RuntimeError) expected action/2 to return a Plug.Conn, all plugs must receive a connection (conn) and return a connection
```

Our controller functions need to return a Plug.Conn, which they also take as a parameter so that's easy enough to fix by making a quick change to our new controller function:


```exs
defmodule Githooker.GithookController do
  use Githooker.Web, :controller
  alias Githooker.EventHandler

  def webhook(conn, _params) do
    conn
  end
end

```

Runniing the tests again gives us another error:

```shell
** (RuntimeError) expected connection to have a response but no response was set/sent.
```

Now we are getting somehwere.  The connection is being returned but no response is set so Phoenix doesnt know what to do.   


Lets try something a bit more useful:
  
    * Grab the github event from the "x-github-event" header
    * Set an error response status on the connection for all github events
    * Send a JSON payload in the response detailing the reason for failing


```exs
defmodule Githooker.GithookController do
  use Githooker.Web, :controller
  alias Githooker.EventHandler

  def webhook(conn, _params) do
    github_event = List.first(get_req_header(conn, "x-github-event"))
    json put_status(conn, 400), %{ status: "Unknown event", event: github_event }
  end
end
```

If we make the changes shown above one of the two tests will start to pass.  The remaining test predictably fails since we are a sending an error response for all github events:

```  ** (RuntimeError) expected response with status 200, got: 400```

To fix this we need to return success for deployment events.  A simple way to achieve this is by adding a case statement to the controller:

```exs
defmodule Githooker.GithookController do
  use Githooker.Web, :controller
  alias Githooker.EventHandler

  def webhook(conn, _params) do
    github_event = List.first(get_req_header(conn, "x-github-event"))

    case {github_event} do
       {"deployment"} ->
         json conn, %{ status: "OK", event: github_event }
       _ ->
         json put_status(conn, 400), %{ status: "Unknown event", event: github_event }
    end
  end
end
```

This will cause both tests to pass but the case statement is pretty ugly.






  Time to refactor:

```
defmodule Githooker.GithookController do
  use Githooker.Web, :controller
  alias Githooker.EventHandler

  def webhook(conn, _params) do
    github_event = List.first(get_req_header(conn, "x-github-event"))
    handleEvent(conn, %{event: github_event})
  end

  def handleEvent(conn, %{event: "deployment"}) do
    json conn, %{ status: "OK", event: "deployment" }
  end

  def handleEvent(conn, %{event: event}) do
    json put_status(conn, 400), %{ status: "Unknown event", event: event }
  end
end
```

This code makes use of Elixir's ability to carry out pattern matching as part of function definitions.  The "handle event" method has a deployment implementation that will only execute if it matches a has containing an event key mapped to a value of "deployment".  It also has a catch all implementation that just extracts the event and uses it set the failure response payload.



















# Next article











#Thinking about logging

It would be useful for Githooker to log all the events it receives. Lets start with some more tests in test/controllers/githook_controller_test.exs:

```
 test "All received GitHub events are logged", %{conn: conn} do
    Logger.configure([level: :info])

    test_events = ~w(
      commit_comment create delete deployment deployment_status fork gollum 
      issue_comment issues member membership page_build public pull_request 
      pull_request_review_comment push release repository status team_add watch 
      some_unknown_event
    )

    Enum.map(test_events, fn (test_event) ->
      logs = capture_log([level: :info],fn ->
        conn
        |> put_req_header("x-github-event", test_event)
        |> post("/api/githook")
      end)

      assert logs =~ "Received GitHub event: #{test_event}"
    end)

    Logger.configure([level: :warn])
  end

  test "Deployment events are logged", %{conn: conn} do
    Logger.configure([level: :info])

    test_event = "deployment"

    logs = capture_log([level: :info],fn ->
      conn
      |> put_req_header("x-github-event", test_event)
      |> post("/api/githook")
    end)

    assert logs =~ "Handling deployment event"

    Logger.configure([level: :warn])
  end

  test "Unknown events are logged", %{conn: conn} do
    Logger.configure([level: :info])

    test_event = "some unknown event"

    logs = capture_log([level: :info],fn ->
      conn
      |> put_req_header("x-github-event", test_event)
      |> post("/api/githook")
    end)

    assert logs =~ "Ignoring unknown event: #{test_event}"

    Logger.configure([level: :warn])
  end


```


Tests fail nicely

Then add these lines to the controller:

```
defmodule Githooker.GithookController do
  use Githooker.Web, :controller
  alias Githooker.EventHandler
  require Logger

  def webhook(conn, _params) do
    github_event = List.first(get_req_header(conn, "x-github-event"))
    Logger.info "Received GitHub event: #{github_event}"
    handleEvent(conn, %{event: github_event})
  end

  def handleEvent(conn, %{event: "deployment"}) do
    Logger.info "Handling deployment event"
    json conn, %{ status: "OK", event: "deployment" }
  end

  def handleEvent(conn, %{event: event}) do
    Logger.info "Ignoring unknown event: #{event}"
    json put_status(conn, 400), %{ status: "Unknown event", event: event }
  end
end
```

And they pass

Tidy up with tags:

```
defmodule Githooker.GithookControllerTest do
  use Githooker.ConnCase
  import ExUnit.CaptureLog

  setup context do
    if context[:loglevel]  do
      Logger.configure([level: context[:loglevel]])

      on_exit fn ->
        Logger.configure([level: :warn])
      end
    end

    :ok
  end

  test "Github deploy events are handled", %{conn: conn} do
    test_event = "deployment"

    result = conn
    |> put_req_header("x-github-event", test_event)
    |> post("/api/githook")

    assert json_response(result, 200) == %{"status" => "OK", "event" => test_event} 
  end

  test "Unknown Github events are not handled", %{conn: conn} do
    test_event = "some_unhandled_event"

    result = conn
    |> put_req_header("x-github-event", test_event)
    |> post("/api/githook")

    assert json_response(result, 400) == %{"status" => "Unknown event", "event" => test_event}
  end

  @tag loglevel: :info
  test "All received GitHub events are logged", %{conn: conn, loglevel: loglevel} do

    test_events = ~w(
      commit_comment create delete deployment deployment_status fork gollum 
      issue_comment issues member membership page_build public pull_request 
      pull_request_review_comment push release repository status team_add watch 
      some_unknown_event
    )

    Enum.map(test_events, fn (test_event) ->
      logs = capture_log([level: loglevel],fn ->
        conn
        |> put_req_header("x-github-event", test_event)
        |> post("/api/githook")
      end)

      assert logs =~ "Received GitHub event: #{test_event}"
    end)
  end

  @tag loglevel: :info
  test "Deployment events are logged", %{conn: conn, loglevel: loglevel} do
    test_event = "deployment"

    logs = capture_log([level: loglevel],fn ->
      conn
      |> put_req_header("x-github-event", test_event)
      |> post("/api/githook")
    end)

    assert logs =~ "Handling deployment event"
  end

  @tag loglevel: :info
  test "Unknown events are logged", %{conn: conn, loglevel: loglevel} do
    test_event = "some unknown event"

    logs = capture_log([level: loglevel],fn ->
      conn
      |> put_req_header("x-github-event", test_event)
      |> post("/api/githook")
    end)

    assert logs =~ "Ignoring unknown event: #{test_event}"
  end
end
```

At this stage the app can be tested using ngrok and a sandbox repo.



#Creating the deployment event handler

First write the test:
CHECK ALL THIS!!!!!!!

```
defmodule Githooker.HttpClientStub do
  def make_request(_, _, _) do
    {:ok, %{
        headers: [status_code: 200]
      }
    }
  end
end

defmodule Githooker.DeploymentHandlerTest do
  use ExUnit.Case
  alias Githooker.DeploymentHandler

  test "trigger deployment" do
    assert {:ok, response} = DeploymentHandler.handle_deployment()
    assert 200 = response[:headers][:status_code]    
  end
end

```



lib/event_handler/deployment_handler.ex
lib/transport/http_client.ex


wiring the client via environment so need to set:

confid/dev.exs
config :githooker, :http_client, Githooker.HttpClient

config/test.exs
config :githooker, :http_client, Githooker.DeploymentHttpClientStub

config/prod.exs
config :githooker, :http_client, Githooker.HttpClient




Bring in httpoison: edit mix.exs
```
 defp deps do
    [{:phoenix, "~> 1.1.4"},
     {:phoenix_html, "~> 2.4"},
     {:phoenix_live_reload, "~> 1.0", only: :dev},
     {:gettext, "~> 0.9"},
     {:cowboy, "~> 1.0"},
     {:httpoison, "~> 0.8.3"}]
  end
```

And add it to applications in mix.exs:
  def application do
    [mod: {Githooker, []},
     applications: [:phoenix, :phoenix_html, :cowboy, :logger, :gettext,
                    :phoenix_ecto, :postgrex,
                    :httpoison
                   ]]
  end


Get the dependencies

```mix deps.get```








defmodule Githooker.DeploymentHandler do
  @http_client Application.get_env(:githooker, :http_client)

  def handle_deployment do
    @http_client.make_request(:post, "http://068d94c5.ngrok.io/api/logger", [])
  end
end





defmodule Githooker.HttpClient do
  use HTTPoison.Base

  def headers do
    %{
      "Content-type" => "application/json",
      "Accept" => " application/json"
    }
  end

  def make_request(:post, url, body) do
    post!(url, body, headers)
  end

end













































add HttPoision to the deps in mix.exs
and also add it to the applications in mix.exs

then run mix deps.get to pull in the dependency
