### Examples

Here are a few examples of effects and handlers you can create

### Complete example from README.md

```javascript
   // create effects
   const authenticatedRequest = effect("authenticatedRequest")
   const getUser = effect("getUser")
   const sendNotification = effect("sendNotification")

  // write your program in direct style using the generator do notation
  const onUserClick = eff(function* () {
     // get the current request from express
     const req = yield authenticatedRequest()

     // await for async call
     const user = yield getUser(req.user.id)

     // throw recoverable exception
     const token = user.token || yield raise("No token found")

     // for each subscriber in the users list of subscribers
     const subscriber = yield forEach(user.subscribers)

     // await for async call
     const result = yield sendNotification({ type: 'clicked', subscriber, details: mouseEvent, user, token })

     return { user, subscriber, result }
  }) // returns [{ user, subscriber1, result1 }, { user, subscriber2, result2 }, ...],
     // the return value depends on how you use the handlers

   // create helper method for fetching
   const fetchApi = (route, method = "GET", body) => fetch("https://www.myapi.com/" + route, {method, body: JSON.stringify(body) }).then(res => res.json())

   // implement the effects using `waitFor` and promises
   const withDataSource = handler({
      getUser: (id) => eff(function* () {
         const user = yield waitFor(() => fetchApi("users/" + id))
         return yield resume(user)
      }),
      sendNotification: (options) => eff(function* () {
         const { subscriber, user } = options;
         const result = yield waitFor(() => fetchApi("users/" + subscriber.id + "/send", "POST", options))
         return yield resume(result)
      })
   })

   // create the handler for the `request` effect that resumes with the request if the user
   // is authenticated, or throws an error if the user is not
   const withAuthenticatedRequest = (request) => handler({
      authenticatedRequest: () => request.user ? resume(request) : raise(new Error("User is not authenticated"))
   })

  app.post("clicked", (req, res) => {
     // since we are handling `withForEach`, the end result gets turned into an array
     // handledProgram: Action<Array<{ user, subscriber, result }>>
     const handledProgram = pipe(
        onUserClick,
        withDataSource,
        withAuthenticatedRequest(req),
        withForEach
     )
     // run the program and await the result
     // note: the program can throw `unhandled handler` and `unhandled exception` 
     const result = await run(handledProgram)
     res.json({
        data: result
     })
  })

```

You could also write your program without using generator functions:

```javascript
  const programPointfree = pipe(
    dependency('auth'),
    chain(auth => pipe(
       cps(window.onclick),
       chain(mouseEvent =>
          pipe(
             getUser(auth.loggedInId),
             chain(user => submitEvent(user, {type: 'clicked', details: mouseEvent}))
          )
       )
    )
  )

  pipe(
    program,
    Effect.do,
    withAuthDependencies,
    withForeach,
    withSubscribe,
    withAsync, // provide promise handler
    run,
    (stream) => stream.subscribe(console.log)
  ) // after each click, logs ['logged with account account1', 'logged with account account2', ...]
```

### Exceptions

You can very easily create try/catch and exceptions with some simple effects

```javascript
// create a new effect
const raise = effect("exn");
```

You can handle raised exceptions using a normal effect handler and simply not resuming the continuation

```javascript
const program = raise(Error('something went wrong')
const myCustomErrorHandler = handler({
    exn: (error) => eff(function* () {
        // we won't resume anything here, instead just return a different answer
        console.log(error)
        return "nothing went wrong! :)"
    })
})
```

Or you can create a handler that takes a function

```javascript
const trycatch = (program) => (oncatch) =>
  handler({
    exn: (error) =>
      eff(function* () {
        // we won't resume, instead we will call the provided function and return its result
        const result = yield oncatch(error);
        return result;
      }),
  })(program);

const handledProgram = trycatch(program)((error) =>
  eff(function* () {
    console.log(error);
    return "nothing went wrong! :)";
  })
);
```

<!--
### Other examples
Hopefully I will add some new examples here soon, but for now you can find some examples of effects and handlers here:

https://github.com/nythrox/effects.js/blob/master/core_effects.js -->
