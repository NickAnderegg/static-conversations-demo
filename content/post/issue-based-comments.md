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
* Providing a system to log in and manage comments, which requires a database backend, is difficult with a static site generator.
* Integrating with an external service (e.g. Disqus) means that you're reliant on an external service to host the content.

Initially, I didn't feel that the benefit of direct engagement outweighed the negative side of comments sections. That is, until I realized an existing tool could be turned into a discussion forum: GitHub Issues.

Over the years, I've seen GitHub repos used to [showcase various one-off demo projects](https://github.com/Wren6991/PicoDVI), and the "Issues" on those projects are often follow-up questions and suggestions for improvement—the same content you might find in a comments section. I decided to use this aspect of GitHub Issues to provide a backend for a comments section for a blog built with a static site generator!

## Implementation

How can this be accomplished? It requires a few tools:

* The Issues tab for this site's repository, where the comments will live.
* The GitHub API, to retrieve that content.
* GitHub Actions, to automatically re-build the site with all of the current comments.
* A lightweight command-line tool to tie all of the pieces together.

This post is going to focus on how I built the CLI that acts as the glue for all of these pieces: an app that I'm calling [Static Conversations](https://github.com/NickAnderegg/static-conversations).

{{< alert title="Links in this post" >}}
All links to the Static Conversations repository in this post will be pinned to the [tree at commit `a6aba5c`](https://github.com/NickAnderegg/static-conversations/tree/a6aba5cb742dfa05dbce93ed629d642e92e16416). The head of the `main` branch of this repo may change significantly over time.
{{</ alert >}}

{{< alert title="Note about code blocks" >}}
Any occurrence of three dots separated by spaces, e.g. "`. . .`" within code blocks indicate code that was removed for brevity.
{{</ alert >}}

### Getting Data from the API

Surprisingly, we only need to call 2-3 API endpoints to get *all* of the data needed for a good comments section! That's because each call to the GitHub API returns a subset of the child nodes of the requested object.

#### `GET`ting the repo's issues

A single `GET` request made to `/repos/{owner}/{repo}/issues` returns not only [a list of the issues associated with a repository](https://docs.github.com/en/rest/reference/issues#list-repository-issues), but also fields like `user` and `labels`, among others. This gives us most of the information we need to render top-level comments. Here is the example response that GitHub provides for this endpoint (with irrelevant fields manually removed):

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

If we want to allow comment replies, users can post comments on the issue that represents a top-level comment. A `GET` request to the `/repos/{owner}/{repo}/issues/{issue_number}/comments` endpoint, substituting the issue's `number` field into the API request path, returns all replies to a top-level comment. Here is a [sample response](https://docs.github.com/en/rest/reference/issues#list-issue-comments) that GitHub provides for this endpoint:

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

Although most endpoints of the API return the child nodes of the requested object, those responses don't contain *all* of the child's fields. In this case, we'll need to make an additional `GET` request to `/users/{username}` for each commenter to get all of the fields we need:

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

These additional fields aren't critical, but they add some polish to the comments section. The `name` and `bio` fields tell readers about the commenter, while the `created_at` field could be used to filter out comments from new accounts.

### Building the CLI Tool

Using [Poetry](https://python-poetry.org/) for package management, we can define our dependencies in the `pyproject.toml` file, and in [just two lines](https://github.com/NickAnderegg/static-conversations/blob/a6aba5cb742dfa05dbce93ed629d642e92e16416/pyproject.toml#L26), specify how the library should be run when called from the command line:

```toml
. . .
[tool.poetry.scripts]
convo = 'convo.console:cli'
. . .
```

Then, `poetry run convo` on the command line will call the `cli()` function present in the `convo.console` module.

#### Structuring the project

When I write any new project from scratch, I always struggle with balancing the principle of ["You Aren't Gonna Need It"](https://martinfowler.com/bliki/Yagni.html) and the dread of having to refactor something in the future if I want to add functionality. The approach I often take—and the approach I've taken here—is to freely add utility modules/classes/functions if they won't result in significant extra work in the short-term, especially if these additions help separate the functional components of the project.

In this case, we have four (sub-)modules:

* **`convo`:** The `__init__.py` implements the `CommentManager` class, which manages and processes all the data we need for our comments section.
* **`convo.api`:** Implements the `GitHubAPI` class, which handles communication with the API.
* **`convo.config`:** Stores configuration for the tool.
* **`convo.console`:** In its current form, this is only a lightweight wrapper that calls methods provided by a `CommentManager` instance.

##### Configuring the app

The simplest module is `convo.config`. Because this tool is intended to run in a GitHub Workflow, we can read everything we need to authenticate with the API from [environment variables](https://docs.github.com/en/actions/reference/environment-variables#default-environment-variables): `GITHUB_ACTOR`, `GITHUB_REPOSITORY`, and [`GITHUB_TOKEN`](https://docs.github.com/en/actions/reference/authentication-in-a-workflow#about-the-github_token-secret).

It [implements a `get_env()`](https://github.com/NickAnderegg/static-conversations/blob/a6aba5cb742dfa05dbce93ed629d642e92e16416/convo/config/__init__.py#L57) method that accepts arbitrary positional arguments specifying which environment variables to read, in order of priority. This way, we can override the `GITHUB_TOKEN` value, for example, by specifying a `CONVO_TOKEN` variable.

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

In general, I prefer to implement any communication with a RESTful API in a class that can handle the boilerplate of preparing requests and processing responses, as I've done [with the `GitHubAPI` class](https://github.com/NickAnderegg/static-conversations/blob/a6aba5cb742dfa05dbce93ed629d642e92e16416/convo/api.py#L17).

First, a few class attributes that we'll use for processing requests and responses, `BASE_USER_AGENT`, `BASE_PATH`, and `UNIVERSAL_KEYS`:

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

And in the class's `__init__` method, several attributes will store the information received from our API requests.

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

I've elected to store the data this way, because it takes advantage of Python objects being passed by reference, rather than by value. Thus, changing a value in one object changes that value in *all* variables which point to that object:

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

This allows us to implement de-duplication of API requests. Remember how the `user` field in issue objects only returns a partial record?

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

We can make a request to retrieve the `octocat` user and store the result as a `dict` under the `octocat` key of the `users` attribute. Then, when we're processing other responses, if we encounter a `user` field, we can check if that user record already exists, and avoid making duplicate requests.

We can see this de-duplication in action in the [`_filter_fields()` method](https://github.com/NickAnderegg/static-conversations/blob/a6aba5cb742dfa05dbce93ed629d642e92e16416/convo/api.py#L103) of `GitHubAPI`:

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

This method filters out irrelevant fields returned by the API. When we call it, we pass `resp`, the JSON response from the API, and `filter_keys`, the set of fields that we want to retain from the result. This set is combined with the `UNIVERSAL_KEYS` class attribute to create the full set of keys to retain.

If the `filtered` dict contains a `user` key, the method passes that username to the [`get_user()` method](https://github.com/NickAnderegg/static-conversations/blob/a6aba5cb742dfa05dbce93ed629d642e92e16416/convo/api.py#L203):

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

If the value passed in the `username` argument is already in the `users` attribute, it returns the existing record. A request to the API is only made if the information we need is missing, avoiding duplicate API requests.

##### Tying it all together

Now, let's take a look at the main `convo` module and the `CommentManager` class within, which ties everything together:

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

Then, we ensure that a `data/` directory exists. The `data/` directory is where Hugo will pull data for rendering comments.

```python
self.working_dir = Path.cwd()

self.output_dir = self.working_dir / "data"
self.output_dir.mkdir(parents=True, exist_ok=True)
```

Finally, we pass the credentials to create a `GitHubAPI` instance that will communicate with the API:

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

The result is a `data/` directory containing the processed data from our API requests. The `data/comments/` directory has an individual JSON file for each comment, named for the `id` returned by the API. The `data/commenters/` directory holds a JSON file for each commenter.

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

Finally, the `comment_mapping.json` file contains a mapping of each comment to the page where it should appear:

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

As mentioned above, the `convo.console` module is a lightweight wrapper that calls the tool's business logic. It is implemented this way to provide a skeleton for future functionality.

Although `console` may be the simplest sub-module, it's also the largest because of a significant amount of boilerplate code that will make it possible to implement powerful logging and debugging features. The most important feature of this module is [the `cli()` method in `convo/console/__init__.py`](https://github.com/NickAnderegg/static-conversations/blob/a6aba5cb742dfa05dbce93ed629d642e92e16416/convo/console/__init__.py#L41), which initializes a `CommentManager`:

```python
def cli(ctx, verbosity, quietness, **kwargs):
    . . .
    ctx.manager = CommentManager()
```

And equally important is [the `cli()` method in `convo/console/commands/load.py`](https://github.com/NickAnderegg/static-conversations/blob/a6aba5cb742dfa05dbce93ed629d642e92e16416/convo/console/commands/load.py#L19), which triggers the `CommentManager` to load comment from the API:

```python
def cli(ctx):
    ctx.manager.load_comments()
```

After we've installed this package in a workflow, we can call `convo load` to load the comment manager, pull data from the API, parse the responses, and write them to filesystem where Hugo expects to find them. Here's the [`process-comments.yml` workflow file from the repository that hosts this site](https://github.com/NickAnderegg/static-conversations-demo/blob/aa4bd0e9670f977fed6738f6167f6d92267eb4ca/.github/workflows/process-comments.yml), which allows this static site to have dynamic comments:

```yaml
name: parse comments from issues

on:
  issues:
    types:
      - opened
      - edited
. . .
jobs:
  process-comments:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ github.token }}

    steps:
      - name: Checkout the repository locally
        uses: actions/checkout@v2

      - name: Install the Static Conversations tool
        run: |
          sudo apt-get install python3.9
          gh release download -R NickAnderegg/static-conversations --pattern '*.whl'
          python3.9 -m pip install ./static_conversations*.whl

        # Where the magic happens
      - name: Load comments
        run: convo load

        # Commit the updated `data/` directory back to this repo and push it
      - name: Commit updated comment files
        run: |
          git config user.name github-actions[bot]
          git config user.email github-actions[bot]@users.noreply.github.com

          git add ./data
          git commit -m "[AUTOMATED] update blog comments"

          git push origin main
```

And if you look just below this paragraph, you should see a fully-funcational comments section generated from the issues on the repository at <https://github.com/NickAnderegg/static-conversations-demo/issues>!
