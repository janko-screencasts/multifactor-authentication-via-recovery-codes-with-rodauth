# Multifactor Authentication via TOTP with Rodauth

## Introduction

Time-based One-Time Password (or TOTP for short), is a standardized algorithm for generating unique numeric passwords using current time as an input.

An application offering TOTP usually presents the user with a QR code, which they can scan into an authenticator app, and this authenticator app then displays generated numeric codes, which are rotated typically every 30 seconds.

Rodauth ships with complete multifactor authentication functionality out-of-the-box, and TOTP is supported as one of the multifactor authentication methods.

I have a Rails app here with Rodauth already setup. If you would like to see how to setup Rodauth from scratch, checkout my [previous video](https://www.youtube.com/watch?v=2hDpNikacf0). Let's add multifactor authentication to this app, allowing users to setup TOTP authentication as the second step of login.

## Installation

Let's start by installing the `rotp` gem, which implements the TOTP algorithm, and `rqrcode` gem, which allows generating QR codes with Ruby. Both these are the dependencies for Rodauth.

```sh
$ bundle add rotp rqrcode
```

Next, we'll create the necessary database table required by Rodauth.

```sh
$ rails generate rodauth:migration otp
# create  db/migrate/20201214200106_create_rodauth_otp.rb

$ rails db:migrate
# == 20201214200106 CreateRodauthOtp: migrating =======================
# -- create_table(:account_otp_keys)
# == 20201214200106 CreateRodauthOtp: migrated ========================
```

In our Rodauth configuration, we can now enable the `otp` feature.

```rb
# app/misc/rodauth_main.rb
class RodauthMain < Rodauth::Rails::Auth
  configure do
    # ...
    enable :otp
  end
end
```

This feature added several endpoints: some for managing TOTP, and other for managing all multifactor authentication methods.

```sh
$ rails rodauth:routes
# ...
#  /multifactor-manage      rodauth.two_factor_manage_path
#  /multifactor-auth        rodauth.two_factor_auth_path
#  /multifactor-disable     rodauth.two_factor_disable_path
#  /otp-auth                rodauth.otp_auth_path
#  /otp-setup               rodauth.otp_setup_path
#  /otp-disable             rodauth.otp_disable_path
```

In our navbar, let's add a link for managing multifactor authentication.

```erb
<!-- app/views/application/_navbar.html.erb -->
<!-- ... -->
         <%= link_to "Manage MFA", rodauth.two_factor_manage_path, class: "dropdown-item" %>
         <div class="dropdown-divider"></div>
<!-- ... -->
```

To make it a bit clearer what's going on, we'll also display the methods the session is currently authenticated by.

```erb
<!-- app/views/application/_navbar.html.erb -->
<!-- ... -->
        <button class="btn btn-info dropdown-toggle" data-bs-toggle="dropdown" type="button">
          Account (<%= rodauth.authenticated_by.join(", ") %>)
        </button>
<!-- ... -->
```

## Setup TOTP

Now when we open our app as a logged in verified user, clicking on the new link we've added takes us straight to the TOTP setup page. A user would now scan this QR code into their authenticator app. For testing purposes, I'll just copy the secret key, and use the command-line interface provided by the `rotp` gem to generate the numeric codes.

```sh
$ rotp --secret 53nurl3dnxxfntnyl7xesl73dvbicsqm
# 430962
```

Once we enter the code, and confirm our password, Rodauth confirms we've now successfully setup TOTP. And we can see that we're also authenticated with the 2nd factor.

Setting up TOTP also created an associated database record, which among other things contains the OTP secret.

```rb
Account.last.otp_key #=>
# #<Account::OtpKey:0x00000001070e6250
#  id: 2,
#  key: "sajxiymzn5bqfq6lgfjjvf5u5pilbyv2",
#  num_failures: 5,
#  last_use: Tue, 05 Jul 2022 13:10:21.244531000 UTC +00:00>
```

## Authenticate via TOTP

That was the setup part. Let's now test the authentication part. When we log out and log back in, we're now required to authenticate with the 2nd factor. Entering the authentication code successfully authenticates us with TOTP, and we can now access the account.

## Additional features

Rodauth sets a default drift of 30 seconds, allowing the client and the server to be slightly out of sync in terms of time, and also allows the user to enter the code shortly after it has expired.

Rodauth also disallows reusing codes, which prevents replay attacks.

If the user enters an invalid code too many times in a row, they'll be locked out of TOTP. This is probably a protection mechanism against bruteforce attacks. In this case, the page is blank, because we don't have an alternative method of multifactor authentication.

## Disable TOTP

When a user with TOTP setup visits the page for managing multifactor authentication, they are redirected to the form for disabling TOTP. Once we confirm the password, and submit the form, Rodauth informs us that TOTP has been disabled, and we're no longer authenticated with the 2nd factor.

Checking the database also confirms that the OTP key record has been deleted.

```rb
Account.last.otp_key #=> nil
```

## Closing words

That's it! We now now have working secure multifactor authentication via TOTP with very little effort.

To reduce the chance users will lock themselves out of their account, it's also recommended to setup an alternative multifactor authentication method, such as recovery codes. We will cover those in the next video.
