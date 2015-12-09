Set up all our mcguffins. Mostly we are just connecting pushover, express, and some glue.

    {spawn, exec} = require 'child_process'
    fs = require 'fs'

    express = require 'express'
    pushover = require 'pushover'
    auth = require 'http-auth'

You'll need to set up your own config file.

    config = require './config'

    app = express()
    repos = pushover config.repo

A simple index page, nothing fancy.

    app.get '/', (req, res) ->
      resp = ["<h1>Current demos</h1>"]
      resp.push '<ul>'
      repos.list (err, list) ->
        resp.push "<li><a href='#{item}'>#{item}</a>" for item in list unless err
        resp.push '</ul>'
        resp.push 'You can clone any demo with git using <code>git clone [url]</code>'
        res.send resp.join '\n'

Then we serve out the static contents of any files under the repo directory.
In theory we could do fancier stuff than this, but let's not for now...

    app.use express.static config.repo

Everything after this point is git http stuff handled by pushover. We don't
want just anyone to be able to update repos, so we put in an auth hook.

    checker = (username, password, cb) -> cb username is config.username and password is config.password
    checkAuth = auth.connect auth.basic realm: "Git Access", checker
    app.use (req, res, next) ->
      if req.query.service is 'git-receive-pack'
        checkAuth req, res, next
      else
        next()

Additionally, pushover is dumb and automatically creates empty repos if you
try to clone them. We stop it from doing this here.

    app.use '/:repo/info/refs', (req, res, next) ->
      if req.query.service is 'git-upload-pack'
        repos.exists req.params.repo, (exists) ->
          if exists
            next()
          else
            res.status(404).send("not found")
      else next()

Here we hook into the actual pushover handler.

    app.use repos.handle.bind(repos)

Pushover has a bug in .list, where .list doesn't work if you call it. We fix
that here.

    repos.list = (cb) -> fs.readdir repos.dirMap(''), cb

Pushover's default .create function theoretically handles non-bare
repositories, but in practice it doesn't really work. We monkey-patch out the
default create so that we can serve the contents back out through express-
static.

    repos.create = (repo, cb=->) ->
      repos.exists repo, (exists) ->
        if exists then next() else repos.mkdir(repo, next)

      next = ->
        dir = repos.dirMap repo;
        exec "git init #{dir} && cd #{dir} && git config receive.denyCurrentBranch updateInstead", (err, stdout, stderr) ->
          if err then cb(stderr or err) else cb(null, stdout)

Some very basic logging.

    repos.on 'fetch', (fetch) ->
      console.log "fetched", fetch.repo, fetch.commit
      fetch.accept()

    repos.on 'push', (push) ->
      console.log "pushed", push.repo, push.commit, push.branch
      push.accept()

And, at last, listen for http requests.

    app.listen process.env.PORT or config.port or 8080