(Core)Zappa
=====

**(Core)Zappa** is a [CoffeeScript](http://coffeescript.org) DSL-ish interface for building web apps on the [node.js](http://nodejs.org) runtime, integrating [express](http://expressjs.com) and other best-of-breed libraries.

    methods = require 'methods'
    invariate = require 'invariate'

Flatten array recursively (copied from Express's utils.js)

    flatten = (arr, ret) ->
      ret ?= []
      for o in arr
        if Array.isArray o
          flatten o, ret
        else
          ret.push o
      ret

Zappa Application
=================

    zappa = {}

Takes in a function and builds express apps based on the rules contained in it.

    zappa.app = ->
      for a in arguments
        switch typeof a
          when 'function'
            func = a
          when 'object'
            options = a

      options ?= {}

      express = options.express ? require 'express'

      context = {zappa,express}

Storage for user-provided stuff:

- Helper functions.

      helpers = {}

The application itself is ExpressJS'.

      app = context.app = express()

Use the `https` options to create a HTTP web server, create a plain HTTP server otherwise.

      http_module = switch
        when options.http_module?
          options.http_module
        when options.https?
          require 'https'
        else
          require 'http'

      context.server = switch
        when options.server?
          options.server
        when options.https?
          http_module.createServer options.https, app
        else
          http_module.createServer app

`apply_helpers`
===============

      apply_helpers = (ctx) ->
        for name, helper of helpers
          do (name, helper) -> ctx[name] = helper
        ctx

route
=====

Register a route with express.

      route = (r) ->
        r.middleware ?= []

        app[r.verb] r.path, r.middleware, (req, res, next) ->

          ctx =
            app: app
            settings: app.settings
            locals: res.locals

            request: req
            req: req
            query: req.query
            params: req.params
            body: req.body
            response: res
            res: res

            send: -> res.send.apply res, arguments
            json: -> res.json.apply res, arguments
            jsonp: -> res.jsonp.apply res, arguments
            redirect: -> res.redirect.apply res, arguments
            format: -> res.format.apply res, arguments

          build_ctx = (o) ->
            _ctx = {}
            _ctx[k] = v for own k,v of ctx
            if o?
              _ctx[k] = v for own k,v of o
            _ctx

          apply_helpers ctx

          try
            await r.handler.call ctx, req, res
          catch error
          if error?
            next error
          else
            next()
          return

Middleware handling
===================

Turn a function such as `route` or `receive` into a middleware-supporting function.

      middlewarify = (handler,verb = null) ->
        (args...) ->
          arity = args.length

Multiple arguments: path, middleware..., handler

          if arity > 1
            handler
              verb: verb
              path: args[0]
              middleware: flatten args[1...arity-1]
              handler: args[arity-1]

Single argument: multiple routes in an object.

          else
            for k, v of args[0]

For each individual entry, if the value is an array, its content must be `middleware..., handler`.

              if v instanceof Array
                handler
                  verb: verb
                  path: k
                  middleware: flatten v[0...v.length-1]
                  handler: v[v.length-1]

Otherwise, the value is simply the handler.

              else
                handler
                  verb: verb
                  path: k
                  handler: v
            return

Verbs (aka HTTP methods)
========================

      for verb in [methods...,'all']
        do (verb) ->
          context[verb] = middlewarify route, verb

.route
======

      context.route = (p) ->
        app.route p

.helper
=======

      context.helper = invariate (k,v) ->
        helpers[k] = v
        return

.engine
=======

      context.engine = invariate (k,v) ->
        app.engine k, v
        return

.set
====

      context.set = invariate (k,v) ->
        app.set k, v
        return

.enable
=======

      context.enable = ->
        app.enable i for i in arguments
        return

.disable
========

      context.disable = ->
        app.disable i for i in arguments
        return

.wrap
=====

Wrap middleware so that they can be ran as regular Express middleware.

      context.wrap = (f) ->
        (req,res,next) ->

This is the context available to Zappa middleware.

          ctx =
            app: app
            settings: app.settings
            locals: res.locals

            request: req
            req: req
            query: req.query
            params: req.params
            body: req.body
            response: res
            res: res

            send: -> res.send.apply res, arguments
            json: -> res.json.apply res, arguments
            jsonp: -> res.jsonp.apply res, arguments
            redirect: -> res.redirect.apply res, arguments
            format: -> res.format.apply res, arguments

          apply_helpers ctx
          try
            await f.call ctx, req, res
          catch error
          if error?
            next error
          else
            next()
          return

.use
====

      context.use = ->

        use = (name, arg = null) ->
          if typeof express[name] is 'function'
            app.use express[name](arg)
          else
            app.use (require name)(arg)

        for a in arguments
          switch typeof a
            when 'function' then app.use a
            when 'string' then use a
            when 'object'
              if a.stack? or a.route? or a.handle?
                app.use a
              else
                use k, v for k, v of a
        return

.settings
=========

      context.settings = app.settings

.locals
=======

      context.locals = app.locals

.include
========

      context.include = (sub,args...) ->
        sub.include.apply context, args

.param
======

      build_param = (callback) ->
        (req,res,next,p) ->

Context available to `param` functions.

          ctx =
            app: app
            settings: app.settings
            locals: res.locals

            request: req
            req: req
            query: req.query
            params: req.params
            body: req.body
            response: res
            res: res

            param: p

          apply_helpers ctx
          try
            await callback.call ctx, req, res, p
          catch error
          if error?
            next error
          else
            next()
          return

      context.param = invariate (k,v) ->
        app.param k, build_param v
        return

.with
=====

Applies a plugin to the current context.

      context.with = invariate (k,v) ->
        ctx = {context,route,require}
        if typeof k is 'string'
          k = require "zappajs-plugin-#{k}"
        k.call ctx, v

Go!
===

      func.apply context

      context

`zappa.run [host,] [port,] [{options},] root_function`
======================================================

Takes a function and runs it as a zappa app. Optionally accepts a port number, and/or a hostname (any order). The hostname must be a string, and the port number must be castable as a number.

    zappa.run = (args...) ->
      host = process.env.ZAPPA_HOST ? null
      port = process.env.ZAPPA_PORT ? 3000
      ipc_path = process.env.ZAPPA_PATH ? null
      root_function = null
      options = {}

      for a in args
        switch typeof a
          when 'string'
            if isNaN( (Number) a ) then host = a
            else port = (Number) a
          when 'number' then port = a
          when 'function' then root_function = a
          when 'object'
            for k, v of a
              switch k
                when 'host' then host = v
                when 'port' then port = v
                when 'path' then ipc_path = v
                else options[k] = v

Listen for connections
----------------------

      listen = (zapp) ->

        {server} = zapp

        server.once 'listening', ->
          addr = server.address()
          channel = if typeof addr is 'string' then addr else addr.address + ':' + addr.port
          console.error """
            Express server listening on #{channel},

          """

          if options.ready?
            options.ready zapp

          return

        switch
          when ipc_path
            server.listen ipc_path
          when host
            server.listen port, host
          else
            server.listen port

      zapp = zappa.app root_function, options
      listen zapp
      zapp

    module.exports = zappa.run
    module.exports.run = zappa.run
    module.exports.app = zappa.app
