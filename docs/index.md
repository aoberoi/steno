# Getting Started with Steno

Steno is your sidekick for developing tests for your Slack app. Use Steno to record and replay HTTP requests and responses to and from your app. Then, you'll be able to run tests on your app without worrying about manually recreating state in actual Slack workspaces (or teams) or manually reproducing events. Hello, automation and continuous integation! :wave:

It doesn't matter which language you chose to program with or how your app is structured. Steno is a CLI tool that starts a server outside your process, so as long as your app speaks HTTP (all Slack apps do), you're ready to go!

## The Workflow

Steno is easiest to use if you follow this workflow. For many of you, this might look familiar- it's based on tried-and-true integration testing patterns. But let's get out of the jargon and jump straight into it.

1. **Pick a behavior in your app that you want to test.**

   Start with something relatively small and self-contained. To illustrate, let's say your app will send a DM to the installing user as soon as it gets installed on a team.

2. **It's time to record**, smile! Just kidding, no cameras please :camera:.

   In **record mode** Steno helps you build **scenarios**, a set of requests and responses that correspond to behaviors or occurrences inside your app. A scenario is represented by a directory inside your project. That directory contains a set of text files; each text file describes one HTTP request and response. The first time you run Steno in record mode, it will create a scenarios folder for you in your working directory.

   Start by choosing a descriptive name for your scenario, e.g. `successfull_installation_will_dm_installing_user`.

   Running `steno record` will prompt Steno to start a local server- by default, `http://localhost:3000/...`. Adjust your code to send requests to that server instead of `https://slack.com/...`.

   Steno will also record requests that are coming **into** your app from the Slack Platform. You may already have a tunneling tool like [ngrok set up](https://api.slack.com/tutorials/tunneling-with-ngrok) to let the Slack Platform reach your app's local server. You can continue to use it, but instead of wiring ngrok up to forward requests directly to a port your app is listening on, substitute the port Steno is listening on- by default, 3010.

   The last piece of wiring we need is to let Steno know where to forward those requests after they've been received locally. For example, if the app is listening on port 5000, the `appBaseUrl` is `localhost:5000`.

   Now we're ready to record. Open your terminal, navigate to a test directory in your project, and launch the tool in record mode:

   `steno record localhost:5000 --scenario-name successfull_installation_will_dm_installing_user`

   Then run your app through the scenario you've named, and Steno will record the requests and responses that are sent and received.

3. With your sidekick Steno :couple: standing by, you can **write your first test.**

   Pick your favorite test runner and write a test case that simulates the behavior you chose. In the case of the DM on install example, you would write a case that completes the OAuth flow for installing your Slack app; your app should behave by exchanging the code for an access token, storing the token, and sending a DM to the installing user. Conclude your test case by asserting that your app is in the state it should be, namely that the token has been stored. But how do we assert that the DM was sent containing the message we intended to be sent? Read on, and we'll find out.

4. Use Steno's **control API** to load scenarios :vhs:.

   Terminate the `steno` command in your terminal and inspect your working directory. You should find that there's a new directory called `scenarios/`. Inside that directory, you should see another directory called `successfull_installation_will_dm_installing_user` which contains the record of all the HTTP interactions your app and Slack just made.

   Next, we'll need to add some setup code to our test case so that Steno is prepared to replay this particular scenario. This can be done by making a request to Steno's control API, which by default is served on port 4000. For example, if you were to add this to your test case using curl, it would look like this:

   `curl -X POST -H "Content-Type: application/json" -d "{ "name":"successfull_installation_will_dm_installing_user" }" http://localhost:4000/start`

   You can adjust this call depending on your language and HTTP client of choice.

5. Steno's control API equips you to answer our burning question :fire:, can we **make an assertion** that our DM was sent?

   With one more request at the end of your test case, you receive data about what actually happened and how it stacks up against our recorded scenario. For example, using curl:

   `curl -X POST http://localhost:4000/stop`.

   You should expect a response back that looks something like this:

   ```json
   {
     "interactions": [
       {
         "direction": "outgoing",
         "request": {
           "timestamp": 1502487343000,
           "method": "POST",
           "url": "/api/oauth.access",
           "headers": {
             "content-type": "application/x-www-form-urlencoded",
             "...": "..."
           },
           "body": "client_id=00000000000.999999999999&client_secret=e9eab23fd04e44c8d1b640e876d39d92&code=00000000000.111111111111.b5fc60d60ec8d65301fcda44028a135bc27cec7ac476938df3b2b15aac73af42&redirect_uri=http%3A%2F%2Fexample.com%2Fcallback"
         },
         "response": {
           "timestamp": 1502487343002,
           "statusCode": 200,
           "headers": {
             "content-type": "application/json; charset=utf-8",
             "...": "..."
           },
           "body": "{\"ok\":true,\"access_token\":\"xoxp-00000000000-11111111111-222222222222-900cf8de83f771f22932027dd9c36dc5\",\"scope\":\"identify,bot\",\"user_id\":\"U11111111\",\"\"bot\":{\"bot_user_id\":\"U00000000\",\"bot_access_token\":\"xoxb-000000000000-I2XejP8axGr15Mz5JHFOKMCe\"}}"
         }
       },
       {
         "direction": "outgoing",
         "request": {
           "timestamp": 1502487343005,
           "method": "POST",
           "url": "/api/chat.postMessage",
           "headers": {
             "content-type": "application/x-www-form-urlencoded",
             "...": "..."
           },
           "body": "token=xoxb-000000000000-I2XejP8axGr15Mz5JHFOKMCe&channel=U11111111&text=Hello%2C%20I%27m%20ExampleBot"
         },
         "response": {
           "timestamp": 1502487343006,
           "statusCode": 200,
           "headers": {
             "content-type": "application/json; charset=utf-8",
             "...": "..."
           },
           "body": "{\"ok\":true,\"channel\":\"D33333333\",\"ts\":\"0000000000.999999\",\"message\":{\"text\":\"Hello, I\'m ExampleBot\",\"username\":\"ExampleBot\",\"bot_id\":\"B44444444\",\"type\":\"message\",\"subtype\":\"bot_message\",\"ts\":\"0000000000.999999\"}}"
         }
       }
     ],
     "meta": {
       "durationMs": 12,
       "unmatchedCount": {
         "incoming": 0,
         "outgoing": 0
       }
     }
   }
   ```

   This represents a comprehensive history of what Steno witnessed since you loaded the scenario with the request to `/start`.

   In our example, we could make a basic assertion that verifies that the last request has a `url` property equal to `"/api/chat.postMessage"`, and that the response's `body` property contains `ok:true`.

6. Pull the plug :electric_plug: and let Steno handle the interactions in **replay mode**.

   Let's go back to the command line and launch steno with a slightly different command:

   `steno replay localhost:5000`

   Run your new test case and watch those assertions succeed :white_check_mark:! (and, if not, maybe that's a good thing- you just caught a bug).

7. Scrub the code and the scenarios of any tokens or secrets :speak_no_evil:, **commit them to your project, rinse and repeat**! Now you can add running Steno in replay mode as a step before starting your test runner in your testing scripts. If you have continuous integration set up, you can rest assured that with every commit along the way you haven't broken your existing behavior :massage:.


**Pro Tip**: This isn't the only way you can use Steno! Even if you don't have access to a specific team or have never been able to produce a specific event, you could write a scenario (using the anticipated request and response bodies), drop it into a scenario directory in your project,and run Steno in replay mode. This will make your app behave as if it magically was talking to the Slack Platform for real :sparkles:. You can hand-edit scenarios based on what's available in our documentation, or take a scenario from someone else who recorded them for you.

## Common Questions

* **Q**: I already use {VCR, node-nock, httpretty, or some other HTTP mocking library for tests}, why should I use Steno?

  **A**: You're a rockstar for thinking about testing from the get go! Those tools are great choices, but Steno still has some advantages for you:
  - Two-way proxying: Most other mocking tools are only concerned with *outgoing* requests from your app. But Slack
    also offers notifications using *incoming* requests (Events API, Interactive Messages, etc.). Steno can handle both.
  - Scenarios are a portable exchange format: No matter which language or toolkit you are using, Steno saves scenarios
    in a format we can all read and write: plain old HTTP. This means you can swap scenario files with other developers
    on your team or in the community, or even attach them when filing bugs.

## Documentation

### Command Line Interface

Steno comes with a `--help` option to help you navigate the command line interface.

```
$ ./steno --help
Commands:
  record <appBaseUrl>  start recording scenarios
  replay <appBaseUrl>  start replaying scenarios

Options:
  --help  Show help                                                    [boolean]
```

Each command also has a `--help` option to get more details.

```
$ ./steno record --help
steno /snapshot/steno/bin/cli.js record <appBaseUrl>

Options:
  --help           Show help                                           [boolean]
  --appBaseUrl     The base URL where all incoming requests from the Slack
                   Platform are targetted. Incoming requestshave the protocol,
                   hostname, and port removed, and are sent to a combination of
                   this URL and the path.                    [string] [required]
  --control-port   The port where the control API is served
                                                      [string] [default: "4000"]
  --in-port        The port where the recording server is listening for requests
                   to forward to the Slack App (where the Slack Platform sends
                   inbound HTTP requests)             [string] [default: "3010"]
  --out-port       The port where the recording server is listening for requests
                   to forward to the Slack Platform (where the Slack App sends
                   outbound HTTP requests)            [string] [default: "3000"]
  --scenario-name  The directory interactions will be saved to or loaded from
                                         [string] [default: "untitled_scenario"]
```

```
$ ./steno replay --help
steno /snapshot/steno/bin/cli.js replay <appBaseUrl>

Options:
  --help           Show help                                           [boolean]
  --appBaseUrl     The base URL where all recorded requests from the replaying
                   server are targetted                      [string] [required]
  --control-port   The port where the control API is served
                                                      [string] [default: "4000"]
  --out-port       The port where the replaying server is listening for requests
                   meant for the Slack Platform (where the Slack App sends
                   outbound HTTP requests)            [string] [default: "3000"]
  --scenario-name  The directory interactions will be saved to or loaded from
                                         [string] [default: "untitled_scenario"]
```

### Control API

See the [control API documentation](./control) for details.

## Known Limitations

* Replay mode will attempt to fire as many incoming requests from the scenario as possible, even in parallel, if all chronologically previous outgoing requests have already been matched. We're investigating a better solution so that you can describe scenarios where those requests should occur serially. Feedback welcome!

* Response URL requests, Incoming Webhooks requests, and any requests to an origin other than 'https://slack.com' do not work yet.

* Request trailers are not currently supported.

* Replaying of chunked encoding requests has not been tested.

## Feedback

Steno is an open source project and we welcome feedback and new contributions. Please leave your comments in [our issue tracker on GitHub](https://github.com/slackapi/steno/issues).
