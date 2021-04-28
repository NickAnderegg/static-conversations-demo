---
author: "Nick Anderegg"
title: "GitHub Issues as a Comments Section"
date: "2021-04-27"
---

When I decided to start blogging again, there was no question that I'd be using [Hugo](https://gohugo.io/) to manage the content. I've used it to build static sites for countless professional projects, and it offers a special combination of power and flexibility that I haven't really found in any other content management software.

<!--more-->

## Motivation

I was uncertain about whether or not to offer a comments section. The biggest argument in favor of comments sections is that they give readers an opportunity to engage directly with content authors. However, the cons abound:

* Comments sections that aren't closely monitored often turn into spam-filled cesspools.
* Providing a system to log in and manage comments, which requires a database backend, is a significant burden for a website built with a static site generator.
* Integrating with an external service (e.g. Disqus) means that you're reliant on a third-party. If the company folds or pivots, all of that content may be lost. Plus, you're piping your readers' information directly to an external party.
* Even a comments section with a strong spam filter can be an attractive target for trolls.

Initially, I didn't feel that the benefit of direct engagement with readers outweighed the negative side of comments sections. That is, until I noticed an existing tool that could be turned into a discussion forum by implementing a lightweight wrapper: GitHub Issues.

Over the years, I've seen GitHub repos used to [showcase various one-off demo projects](https://github.com/Wren6991/PicoDVI), and the `Issues` tab on those projects often fills with follow-up questions and suggestions for improvement—the same type of content you might find in a dedicated comments section. It occurred to me that this aspect of GitHub Issues could be intentionally harnessed to provide a comments section for a blog built with a static site generator!

## Implementation

How can this be accomplished? It only requires a few tools:

* The Issues section for this site's repository, where the content will live.
* The GitHub API, which can retrieve that content.
* GitHub Actions, to automatically re-build the site with all of the current comments.
* A lightweight command-line interface to tie all of the pieces together.

This post is going to focus on how I built the CLI that acts as the glue for all of these pieces: an app that I'm calling [Static Conversations](https://github.com/NickAnderegg/static-conversations).

{{< alert title="Links in this post" >}}
All links to the Static Conversations repository in this post will be pinned to the [tree at commit `88dfb24`](https://github.com/NickAnderegg/static-conversations/tree/88dfb2441c03d8f9ee2f6ef338bb237a81a9e603). The head of the `main` branch of this repo may change significantly over time.
{{</ alert >}}

{{< alert title="Note about code blocks" >}}
Any occurrence of three dots separated by spaces, e.g. "`. . .`" within code blocks indicate code that was removed for brevity. In many places, docstrings and comments have also been removed from code examples without indication.
{{</ alert >}}

### Getting Data from the API

Surprisingly, we only need to call 2-3 API endpoints to get *all* of the data necessary to implement a full-featured comment management tool! A big reason for this is that each call to the GitHub API returns a significant number of the nodes that connect to the object returned by each endpoint.

#### `GET`ting the repo's issues

A single `GET` request made to `/repos/{owner}/{repo}/issues` returns not only [a list of the issues associated with a repository](https://docs.github.com/en/rest/reference/issues#list-repository-issues), but also an object representing the most important data of the `user` and `labels` fields, among other. This provides us with most of the information we need to render top-level comments. Here is the example response that GitHub provides for this endpoint (with the fields that are not relevant to our use case manually removed):

```json
[
  {
    "id": 1,
    "node_id": "MDU6SXNzdWUx",
    "html_url": "https://github.com/octocat/Hello-World/issues/1347",
    "number": 1347,
    "state": "open",
    "title": "Found a bug",
    "body": "I'm having a problem with this.",
    "user": {
      "login": "octocat",
      "id": 1,
      "node_id": "MDQ6VXNlcjE=",
      "avatar_url": "https://github.com/images/error/octocat_happy.gif",
      "html_url": "https://github.com/octocat"
    },
    "labels": [
      {
        "id": 208045946,
        "node_id": "MDU6TGFiZWwyMDgwNDU5NDY=",
        "url": "https://api.github.com/repos/octocat/Hello-World/labels/bug",
        "name": "bug",
        "description": "Something isn't working",
        "color": "f29513",
        "default": true
      }
    ],
    "locked": true,
    "active_lock_reason": "too heated",
    "comments": 0,
    "closed_at": null,
    "created_at": "2011-04-22T13:33:48Z",
    "updated_at": "2011-04-22T13:33:48Z",
    "author_association": "COLLABORATOR"
  }
]
```

#### Polling for replies

If we want to allow our commenters to reply to other comments, they can do so by posting comments on the issues that represent the top-level comments. A `GET` request to the `/repos/{owner}/{repo}/issues/{issue_number}/comments` endpoint, substituting the issue's `number` field for `{issue_number}` in the API request path, returns all replies to a top-level comment. Here is the [sample response](https://docs.github.com/en/rest/reference/issues#list-issue-comments) that GitHub provides for this endpoint, again with irrelevant fields removed:

```json
[
  {
    "id": 1,
    "node_id": "MDEyOklzc3VlQ29tbWVudDE=",
    "html_url": "https://github.com/octocat/Hello-World/issues/1347#issuecomment-1",
    "body": "Me too",
    "user": {
      "login": "octocat",
      "id": 1,
      "node_id": "MDQ6VXNlcjE=",
      "avatar_url": "https://github.com/images/error/octocat_happy.gif",
      "html_url": "https://github.com/octocat"
    },
    "created_at": "2011-04-14T16:00:49Z",
    "updated_at": "2011-04-14T16:00:49Z",
    "author_association": "COLLABORATOR"
  }
]
```

#### Learning more about each commenter

Although most endpoints of the GitHub API return the nodes connected to the object returned by each endpoint, those responses don't necessarily contain *all* of the fields present in the connected nodes. For our particular use case, we'll need to make an additional `GET` request to `/users/{username}` for each commenter. Unfortunately, this request to get the four additional fields relevant to us (`name`, `bio`, `created_at`, and `updated_at`) requires GitHub to return an object with 23 irrelevant fields. Those have been removed from the example response presented here:

```json
{
  "login": "octocat",
  "id": 1,
  "node_id": "MDQ6VXNlcjE=",
  "avatar_url": "https://github.com/images/error/octocat_happy.gif",
  "html_url": "https://github.com/octocat",
  "name": "monalisa octocat",
  "bio": "There once was...",
  "created_at": "2008-01-14T04:33:35Z",
  "updated_at": "2008-01-14T04:33:35Z"
}
```

These fields aren't critical, but they add some polish to our comments section. The `name` and `bio` fields let us show who is commenting, while the `created_at` and `updated_at` fields are useful if we want to implement some rudimentary filtering, such as filtering out comments from brand new accounts.

### Building the CLI Tool

Modern Python tools like [Poetry](https://python-poetry.org/) make it much easier to build and run a CLI than it has been in the past. Once we define our dependencies in the `pyproject.toml` file, [just two lines](https://github.com/NickAnderegg/static-conversations/blob/88dfb2441c03d8f9ee2f6ef338bb237a81a9e603/pyproject.toml#L26) allow us to specify how the library should be run when called from the command line:

```toml
. . .
[tool.poetry.scripts]
convo = 'convo.console:cli'
. . .
```

Then when we run `poetry run convo` from the command line, Poetry will call the `cli()` function present in the `convo.console` module.

#### Structuring the project

When I write any new project from scratch, I always struggle with balancing the principle of ["You Aren't Gonna Need It"](https://martinfowler.com/bliki/Yagni.html) and the dread of having to refactor something in the future if I want to change even the most basic functionality. The approach I often take to this problem—and the approach I've taken here—is to freely add utility modules/classes/functions in any areas where it won't result in significant extra work in the short-term, especially if these additions just separate the various functional components of the project.

In this case, we have four (sub-)modules:

* **`convo`:** The `__init__.py` implements the `CommentManager` class, which manages and processes all the data we need for our comments section.
* **`convo.api`:** This module implements the `GitHubAPI` class, which manages communication with the GitHub API.
* **`convo.config`:** A module that stores configuration for the tool. At this stage, the tool's configuration is read from environment variables.
* **`convo.console`:** In its current form, this is essentially just a lightweight wrapper that calls methods provided by a `CommentManager` instance.

##### Configuring the app

The simplest of these four modules is `convo.config`. Because this tool is intended to run in a GitHub Workflow, we can read get everything we need to authenticate with the API from [environment variables provided by the runner environment](https://docs.github.com/en/actions/reference/environment-variables#default-environment-variables). The variables we need to read are `GITHUB_ACTOR`, to determine the user which we will use to authenticate; `GITHUB_REPOSITORY`, which tells us where our workflow is running, and `GITHUB_TOKEN`; which we will use [to authenticate with the API](https://docs.github.com/en/actions/reference/authentication-in-a-workflow#about-the-github_token-secret).

We also [implement a `get_env()`](https://github.com/NickAnderegg/static-conversations/blob/main/convo/config/__init__.py#L57) static method that allows us to pass an arbitrary number of positional arguments specifying which environment variables to read, in order of priority. This way, if we have some reason to override the default `GITHUB_TOKEN` value, we can specify a `CONVO_TOKEN` or `GH_TOKEN` variable with the override value.

```python
. . .
class ConvoConfig(object):
    def __init__(self):
        . . .
        token_vars = ["CONVO_TOKEN", "GH_TOKEN", "GITHUB_TOKEN"]
        token = self.get_env(*token_vars)
        if token is None:
            vars_string = "`, `".join(token_vars)
            error_str = f"An auth token must be specified in one of these environment variables: `{vars_string}`"
            raise RuntimeError(error_str)
        . . .

    @staticmethod
    def get_env(*keys: str, default: t.Any = None) -> t.Union[str, t.Any]:
        for key in keys:
            val = os.getenv(key)
            if val is not None and val != "":
                return val

        return default
```

##### Communicating with the API

In general, I prefer to implement any communication with a RESTful API in a class that can handle the boilerplate of preparing requests and processing the responses, and that's exactly what I've done here [with the `GitHubAPI` class](https://github.com/NickAnderegg/static-conversations/blob/main/convo/api.py#L17).

First, we define a few class attributes that we'll use for building and processing requests, `BASE_USER_AGENT`, `BASE_PATH`, and `UNIVERSAL_KEYS`:

```python
. . .
class GitHubAPI(object):

    BASE_USER_AGENT: str = f"StaticConversations/{__version__}"
    BASE_PATH: str = "https://api.github.com"
    UNIVERSAL_KEYS: t.Set[str] = {
        "id",
        "node_id",
        "html_url",
        "user",
        "created_at",
        "updated_at",
    }
```

And in the class's `__init__` method, we specify several instance attributes that store the information we need to prepare requests and `dict`s that store data from responses.

```python
    def __init__(self, /, credentials: dict[str, str]) -> None:
        self.credentials: dict[str, str] = credentials
        self.repo: str = credentials["repo"]

        self.session: requests.Session = requests.Session()

        self.nodes: dict[str, dict] = {}
        self.users: dict[str, dict] = {}
        self.issues: dict[str, dict] = {}
        self.issue_comments: dict[str, dict] = {}

        self.headers: dict[str, str] = {
            "Accept": "application/vnd.github.v3+json",
            "User-Agent": f"{self.BASE_USER_AGENT} ({sys.platform})",
            "Content-Type": "application/json; charset=utf-8",
            "Authorization": f"token {self.credentials['auth']}",
        }
```

The three attributes for storing response data, `users`, `issues`, and `issue_comments`, represent the three data types that we will be requesting from the API, with `nodes` storing *every* returned object, keyed by its `node_id` value.

I've elected to store the data returned from the API this way, because it takes advantage of the fact that all objects in Python are passed/assigned by reference, rather than by value. When you assign an object, such as a class instance or `dict` to multiple variables, Python assigns a pointer to the object, rather than the object itself. Thus, changing a value in one object changes means the change is reflected in all variables which point to this object:

```python
>>> primary_dict = {"id": 12345, "name": "Bob",}
>>> x = primary_dict
>>> y = primary_dict
>>> z = primary_dict

>>> print(z)
{'id': 12345, 'name': 'Bob'}

>>> z["name"] = "Alice"

>>> print(primary_dict)
{'id': 12345, 'name': 'Alice'}
```

Why is this useful? It allows us to implement de-duplication of the values returned by the API. Remember how the `user` field in issue objects only returns a partial record?

```json
[
  {
    "title": "Found a bug",
    "body": "I'm having a problem with this.",
    "user": {
      "login": "octocat",
      "id": 1,
      "node_id": "MDQ6VXNlcjE=",
      "avatar_url": "https://github.com/images/error/octocat_happy.gif",
      "html_url": "https://github.com/octocat"
    },
```

After we make a request to the API to retrieve the full object for the `octocat` user, we can store the resulting object as a `dict` under the `octocat` key within the instance's `users` attribute. Then, when we're processing other issues and issue comments, if we encounter a `user` field, we can check if that username exists as a key in the `users` attribute, and avoid making duplicate requests.

We can see this de-duplication in action in the [`_filter_fields()` method](https://github.com/NickAnderegg/static-conversations/blob/main/convo/api.py#L103) of the `GitHubAPI` class:

```python
    def _filter_fields(
        self,
        resp: dict[str, t.Any],
        filter_keys: set,
    ) -> dict[str, t.Any]:

        filtered = {
            key: val
            for key, val in resp.items()
            if key in (self.UNIVERSAL_KEYS | filter_keys)
        }

        if "user" in filtered.keys():
            user = self.get_user(filtered["user"]["login"])
            filtered["user"] = user

        return filtered
```

The primary purpose of the above method is to filter out irrelevant fields that are returned by the GitHub API. When we call this method, we pass `resp`, the raw JSON response from the API, and `filter_keys`, the set of object keys that we want to retain from the result. This set is combined with the `UNIVERSAL_KEYS` class attribute discussed previously to create the full set of keys to retain.

Within this method, we create a new variable called `filtered`, which is a `dict` containing only the key-value pairs specified in the set of keys to retain. Following that, it checks to see if the `filtered` dict contains a `user` key, and if it does, passes the username to the [`get_user()` method](https://github.com/NickAnderegg/static-conversations/blob/main/convo/api.py#L203), shown here:

```python
    def get_user(self, username):
        if username in self.users:
            return self.users[username]

        resp = self.get_resource(["users", username])

        user = self._filter_user(resp)

        self.nodes[user["node_id"]] = user
        self.users[user["login"]] = user

        return user
```

The first thing this method does is check to see if the value passed in the `username` argument is present in the instance's `users` attribute. If it is already present, it returns the object we've already retrieved, only making a new request to the API if the information for that user has not already been retrieved. This avoids making repeated requests to the GitHub API for the same information.

##### Tying it all together

Now, let's take a look at the main `convo` module and the `CommentManager` class within, which implements the business logic that ties everything together:

```python
class CommentManager(object):
    def __init__(self):

        config = ConvoConfig()
        self.credentials = config.credentials

        self.working_dir = Path.cwd()

        self.output_dir = self.working_dir / "data"
        self.output_dir.mkdir(parents=True, exist_ok=True)

        self.api = GitHubAPI(credentials=self.credentials)
```

First, we instantiate the `ConvoConfig` class, which will read the app's configuration from environment variables, and assign it to the manager's `credentials` attribute:

```python
config = ConvoConfig()
self.credentials = config.credentials
```

Then, we check the working directory from which the app is being run and ensure that a `data/` directory exists. The `data/` directory is where Hugo will pull data for rendering each post's comments section.

```python
self.working_dir = Path.cwd()

self.output_dir = self.working_dir / "data"
self.output_dir.mkdir(parents=True, exist_ok=True)
```

Finally, we pass the credentials to create a `GitHubAPI` instance that we'll use to communicate with the GitHub API:

```python
self.api = GitHubAPI(credentials=self.credentials)
```

We're ready to go! A call to the `load_comments()` method of the manager object triggers the sequence of calls that process and write out all of our data:

```python
    def load_comments(self):
        print("Loading comments...")

        # Make a call to `get_issues()` of our GitHubAPI instance, which pulls
        # the list of issues attached to our repo.
        issues = self.api.get_issues()

        # We can just directly access `api.users` because calling the `api.get_issues()`
        # method automatically populates the `users`, `nodes`, `issues`, and
        # `issuse_comments` attributes of the GitHubAPI instance.
        users = self.api.users
        self._process_users(users)

        # Do all the post-processing we need to render comments when Hugo
        # regenerates the site.
        comments, comment_mapping = self._parse_comments(issues)

        # Write the list of comments out to individual files in the `data/` directory.
        self._render_comments(comments)

        # Write a file containing the mapping between posts and top-level comments,
        # which our Hugo template will read to determine which comments to render.
        mapping_file = self.output_dir / "comment_mapping.json"
        with mapping_file.open("w", encoding="utf-8") as f:
            json.dump(comment_mapping, f, indent=4)
```

The end result here is a `data/` directory containing all of the processed data from our API requests. The `data/comments/` directory contains individual JSON files representing each top-level comment (GitHub's issues) and each reply (GitHub's comments on issues), named for the `id` field returned by the API. The `data/commenters/` directory holds a single JSON file for each commenter.

```
❯ tree data
data
├── comment_mapping.json
├── commenters
│   └── octocat.json
└── comments
    ├── 827091484.json
    ├── 827091907.json
    ├── 868074380.json
    └── 868075880.json
```

Finally, the `comment_mapping.json` file contains a JSON object which maps each top-level comment to the page which is should appear under:

```
❯ cat data/comment_mapping.json
{
    "post/some-blog-post": [
        868075880,
        868074380
    ]
}
```

## Usage

As mentioned above, the `convo.console` module is nothing more than a lightweight wrapper that calls the classes that do the actually processing. It has been implemented this way because it provides a skeleton structure that will mean that future expansion will require minimal refactoring.

Although `console` may be the simplest sub-module, but it's also the largest because of a significant amount of boilerplate code in place that will make complex logging and debugging simpler to implement. The most important feature of this module is [the `cli()` method in `convo/console/__init__.py`](https://github.com/NickAnderegg/static-conversations/blob/main/convo/console/__init__.py#L41), which initializes a `CommentManager`:

```python
def cli(ctx, verbosity, quietness, **kwargs):
    . . .
    ctx.manager = CommentManager()
```

And equally important is [the `cli()` method in `convo/console/commands/load.py`](https://github.com/NickAnderegg/static-conversations/blob/main/convo/console/commands/load.py#L19), which triggers the `CommentManager` to load comment from the API:

```python
def cli(ctx):
    ctx.manager.load_comments()
```

After we've installed this package using `poetry install`, we can call `poetry run convo load` to load the comment manager, pull the relevant data from the GitHub API, parse the responses, and write them out to filesystem where Hugo expects to find them.
