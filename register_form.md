# Register form

Register feature will be based on a standard form, so let's add one to the `Form` module:

```fsharp
open System.Net.Mail
```

```fsharp
type Register = {
    Username : string
    Email : MailAddress
    Password : Password
    ConfirmPassword : Password
}

let pattern = @"(\w){6,20}"

let passwordsMatch = 
    (fun f -> f.Password = f.ConfirmPassword), "Passwords must match"

let register : Form<Register> = 
    Form ([ TextProp ((fun f -> <@ f.Username @>), [ maxLength 30 ] )
            PasswordProp ((fun f -> <@ f.Password @>), [ passwordRegex pattern ] )
            PasswordProp ((fun f -> <@ f.ConfirmPassword @>), [ passwordRegex pattern ] )
            ],[ passwordsMatch ])
```

In the above snippet:

- we open the `System.Net.Mail` namespace to use the `MailAddress` type
- the form consists of 4 fields:
    - `Username` of `string`
    - `Email` of type `MailAddress` (ensures proper validation of the field)
    - `Password` of type `Password` (ensures proper HTML input type)
    - `ConfirmPassword` of the same type
- `pattern` is a regular expression pattern for password
- `passwordsMatch` is a server-side only validation function
- `register` is the definition of our form, with a few constraints:
    - `Username` must be at most 30 characters long
    - `Password` must match the regular expression
    - `ConfirmPassword` must match the regular expression

Server-side only validation, like `passwordMatch` are of type `('FormType -> bool) * string`.
So this is just a tuple of a predicate function and a string error.
We can create as many validations as we like, and pass them to the `Form` definition.
These can be used for validations that lookup more than one field, or require some complex logic.
We won't create client-side validation to check if the passwords match in the tutorial, but it could be achieved with some custom JavaScript code.

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