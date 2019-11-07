    ({expect} = require 'chai').should()
    request = require 'superagent'
    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'Zappa', ->
      port = 3000
      Zappa = require '..'
      it 'should start a service', ->
        p = port++
        {server} = Zappa p, ->
          @get '/', -> @json ok:true

        after -> server.close()

        res = await request.get "http://127.0.0.1:#{p}/"
        res.should.have.property 'body'
        res.body.should.have.property 'ok', true

      it 'should handle middleware', ->
        p = port++
        {server} = Zappa p, ->
          mw = @wrap -> @req.bear = 42
          @get '/foo', mw, -> @json bear: @req.bear

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/foo"
        res.should.have.property 'body'
        res.body.should.have.property 'bear', 42

      it 'should support `done`', ->
        p = port++
        called_me = false
        {server} = Zappa p, ->
          mw1 = @wrap ->
            @res.json kitty: 42
            @res.end()
            @done()
            return
          mw2 = @wrap -> called_me = true
          @get '/foo', mw1, mw2, -> @json bear: 43

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/foo"
        res.should.have.property 'body'
        res.body.should.have.property 'kitty', 42
        expect(called_me).to.be.false

      it 'should handle async middleware', ->
        p = port++
        {server} = Zappa p, ->
          mw = @wrap ->
            @req.bear = 43
            await sleep 100
          @get '/foo', mw, -> @json bear: @req.bear

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/foo"
        res.should.have.property 'body'
        res.body.should.have.property 'bear', 43

      it 'should provide `use`', ->
        p = port++
        {server} = Zappa p, ->
          mw = @wrap -> @req.bear = 54
          @use mw
          @get '/bear', -> @json bear: @req.bear

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/bear"
        res.should.have.property 'body'
        res.body.should.have.property 'bear', 54

      it 'should provide `use` for async', ->
        p = port++
        {server} = Zappa p, ->
          mw = @wrap ->
            @req.bear = 55
            await sleep 100
          @use mw
          @get '/bear', -> @json bear: @req.bear

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/bear"
        res.should.have.property 'body'
        res.body.should.have.property 'bear', 55

      it 'should accept parameters', ->
        p = port++
        {server} = Zappa p, ->
          @get '/ponchos/:id', -> @json ponchos: parseInt @params.id, 10

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/ponchos/4"
        res.should.have.property 'body'
        res.body.should.have.property 'ponchos', 4

      it 'should provide static helpers', ->
        p = port++
        {server} = Zappa p, ->
          @helper cfg: here: 'France'
          @get '/cfg/:field', -> @json @cfg[@params.field]

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/cfg/here"
        res.should.have.property 'body', 'France'

      it 'should provide dynamic helpers', ->
        p = port++
        {server} = Zappa p, ->
          @helper map: ->
            [@params.field]: 'France'
          @get '/cfg/:field', -> @json @map()

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/cfg/there"
        res.should.have.property 'body'
        res.body.should.have.property 'there', 'France'

      it 'should provide `include`', ->
        p = port++
        {server} = Zappa p, ->
          @include include: ->
            @get '/included', -> @json true
          @get '/master', -> @json false

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/included"
        res.should.have.property 'body', true
        res = await request.get "http://127.0.0.1:#{p}/master"
        res.should.have.property 'body', false

      it 'should support `route`', ->
        p = port++
        {server} = Zappa p, ->
          @get '/:id', ->
            if @params.id is 'here'
              @json 'here'
            else
              'route'

          @get '/:id', ->
            @json 'there'

        after -> server.close()
        res = await request.get "http://127.0.0.1:#{p}/here"
        res.should.have.property 'body', 'here'
        res = await request.get "http://127.0.0.1:#{p}/anywhere"
        res.should.have.property 'body', 'there'

      it 'should allow logging', ->
        p = port++
        requests = 0
        errors = 0
        {server} = Zappa p, ->
          @use (req,res,next) ->
            requests++
            await next()
            errors++ if res.statusCode >= 400

          @get '/good', -> @json yes

        after -> server.close()
        try await request.get "http://127.0.0.1:#{p}/good"
        expect(requests).to.eql 1
        expect(errors).to.eql 0
        try await request.get "http://127.0.0.1:#{p}/bad"
        expect(requests).to.eql 2
        expect(errors).to.eql 1
