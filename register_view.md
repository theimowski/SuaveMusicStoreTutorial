# Register view

With the form definition in place, let's proceed to `View`:

```fsharp
let register msg = [
    h2 "Create a New Account"
    p [
        text "Use the form below to create a new account."
    ]
    
    divId "register-message" [
        text msg
    ]

    renderForm
        { Form = Form.register
          Fieldsets = 
              [ { Legend = "Create a New Account"
                  Fields = 
                      [ { Label = "User name (max 30 characters)"
                          Xml = input (fun f -> <@ f.Username @>) [] }
                        { Label = "Email address"
                          Xml = input (fun f -> <@ f.Email @>) [] }
                        { Label = "Password (between 6 and 20 characters)"
                          Xml = input (fun f -> <@ f.Password @>) [] }
                        { Label = "Confirm password"
                          Xml = input (fun f -> <@ f.ConfirmPassword @>) [] } ] } ]
          SubmitText = "Register" }
]
```

As you can see, we're using the `msg` parameter here similar to how it was done in `View.logon` to include possible error messages.
The rest of the snippet is rather self-explanatory.