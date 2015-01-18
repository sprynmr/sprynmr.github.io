- signal of signals
- RACCommand
    - Allows for creation of signals
    - Returns a signal from the `execute` command
    - Wraps a signal with some nicities like .executing
    - Good if you need a contract, or to prevent multiple executions
    - Prevents the multiple subscriber issue found above
    - .executionSignals is a signal of signals
        - explain `switchToLatest`?

- Model changes must go through the view model
- Binding data from parent view-models to child view-models
- How does a button in a subview trigger something on the main view-model?
    - Responder chain is helpful
    - Potentially calling up through the view-model chain, but feels weird
    - Typical delegates and block callbacks