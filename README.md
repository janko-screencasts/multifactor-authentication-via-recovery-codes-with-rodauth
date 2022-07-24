# Multifactor Authentication via Recovery Codes with Rodauth

## Introduction

In the previous video, we've added multifactor authentication via TOTP to this Rails application.

Now, if the user happens to lose access to their TOTP application, or they get locked out of TOTP, they could risk locking themselves out of their account.

To help with that, Rodauth provides the [`recovery_codes`](http://rodauth.jeremyevans.net/rdoc/files/doc/recovery_codes_rdoc.html) feature, which allows generating a set of single-use codes for the user, which they can use with the 2nd factor instead of TOTP. Let's set this up in our application.

## Installation

We'll start by creating the necesssary database table:

```sh
$ rails generate rodauth:migration recovery_codes
# create  db/migrate/20201214200106_create_rodauth_recovery_codes.rb

$ rails db:migrate
# == 20201217071036 CreateRodauthRecoveryCodes: migrating =======================
# -- create_table(:account_recovery_codes, {:primary_key=>[:id, :code]})
# == 20201217071036 CreateRodauthRecoveryCodes: migrated ========================
```

Then, in our Rodauth configuration, we'll enable the `recovery_codes` feature:

```rb
# app/misc/rodauth_main.rb
class RodauthMain < Rodauth::Rails::Auth
  configure do
    # ...
    enable :otp, :recovery_codes
  end
end
```

This added routes for viewing and adding recovery codes, as well as for authenticating via a recovery code:

```sh
$ rails rodauth:routes
# ...
#  /recovery-auth           rodauth.recovery_auth_path
#  /recovery-codes          rodauth.recovery_codes_path
```

## Setup recovery codes

TOTP is still considered the primary multifactor authentication method, so we'll first need to set it up before we can access recovery codes.

When we again visit the page for managing multifactor authentication methods, we can now see a new link for viewing recovery codes. When we click on the link, and confirm our password, we're greeted with a blank page. This is because, by default, Rodauth doesn't automatically generate any recovery codes. We can create them by submitting this form.

Looking at our database, we can see that each recovery code was saved in a separate table row.

```rb
Account.first.recovery_codes
# #=> [#<Account::RecoveryCode:0x000000011025cf68 id: 1, code: "-pOawxfBw95id623t0DQHzVPKc7rTwngC2oPeORQ5pw">,
#      #<Account::RecoveryCode:0x0000000110284c98 id: 1, code: "30GRJkr1BheZztvFZcDeRSNy6yhzigXH6zB-yvzP4Io">,
#      #<Account::RecoveryCode:0x0000000110284ab8 id: 1, code: "3vfVsss3W_KAMUVPCVgYxwBCANLWVDpSC_IOs_ACMeM">,
#      #<Account::RecoveryCode:0x0000000110284928 id: 1, code: "4Z_DoDGv0nFDdWHNAp6oX9o0wf6_dJF51-FZWjwJ6as">,
#      #<Account::RecoveryCode:0x0000000110284798 id: 1, code: "DS6dtRNnvzSCWzm8jg4lltOzBE5vTN_xflNdToIPw3A">,
#      #<Account::RecoveryCode:0x0000000110284608 id: 1, code: "DSGysPPN9bcg5fJBBqavyfJ3pABvslD760z_DFR1wX4">,
#      #<Account::RecoveryCode:0x0000000110284478 id: 1, code: "ERCUrIjGtJpJ0ybb6mNikpO_0r6pEyJ97LPvza6KXDw">,
#      #<Account::RecoveryCode:0x00000001102842e8 id: 1, code: "FCChtnh8oJprccZlT3-2helLhUfkmtLO9zInCIG4isk">,
#      #<Account::RecoveryCode:0x0000000110284158 id: 1, code: "FumQDUIqHkr7bBZz5TwOru7R-LLOwYZwwQmV7Y4yny0">,
#      #<Account::RecoveryCode:0x000000011027fc70 id: 1, code: "H6FUlLQj2_y8iqNPK5AUmfg0NeG3sEL2vVe5lsuwMX8">,
#      #<Account::RecoveryCode:0x000000011027fa18 id: 1, code: "L2Y-lHcGAPX8eqcq0dZP9aKBUjJNq1qE8vjT-V2BUoM">,
#      #<Account::RecoveryCode:0x000000011027f658 id: 1, code: "LcUGvbkgy91r6GY2sdcxgjMFMx8K7e_Ln_NykUljkeE">,
#      #<Account::RecoveryCode:0x000000011027f298 id: 1, code: "ZDfv5czC8okh_12ZzI8GG7RwOexAdJeqSc5_Z9rpe34">,
#      #<Account::RecoveryCode:0x000000011027f108 id: 1, code: "gIPjsHfXCQKQ55KLw-CbJPY5-v29TRvTjtoQF07cUHM">,
#      #<Account::RecoveryCode:0x000000011027ef50 id: 1, code: "u1yOol7vZf7TaNaRgEAP4Zb4bugZMOFyGaTAgZq6V80">,
#      #<Account::RecoveryCode:0x000000011027ed70 id: 1, code: "y2_N79Vt_m6FYnRiG_vBKJ8wUlENECrDjcJYoQ_muMs">]
```

## Authenticate via recovery code

Let's copy one of these recovery codes, and log out, and then log back in. In addition to the authenticating via TOTP, Rodauth now offers us to also authenticate via a recovery code. When we enter the recovery code we copied into the clipboard, Rodauth confirms we're now successfully multifactor authenticated.

Recovery codes are single-use, so in our Rails logs we'll see that Rodauth has deleted the code we've just used for authentication.

```sql
DELETE FROM "account_recovery_codes" WHERE (("id" = 1) AND ("code" = '30GRJkr1BheZztvFZcDeRSNy6yhzigXH6zB-yvzP4Io'))
```

## Autogenerate recovery codes

Great, that works! However, with the current design, recovery codes are not easily discoverable and are optional, which means very few people will set them up. Let's make it so that recovery codes are automatically generated on TOTP setup, and are immediately presented to the user.

In our Rodauth configuration, we'll configure recovery codes to automatically generate whenever another multifactor authentication method is enabled. When the last multifactor authentication method has been disabled, we'll delete recovery codes.

```rb
# app/misc/rodauth_main.rb
class RodauthMain < Rodauth::Rails::Auth
  configure do
    # ...
    auto_add_recovery_codes? true
    auto_remove_recovery_codes? true
  end
end
```

Next, after TOTP setup, we'll render the generated recovery codes, and also ask the user the save them.

```rb
# app/misc/rodauth_main.rb
class RodauthMain < Rodauth::Rails::Auth
  configure do
    # ...
    after_otp_setup do
      set_notice_now_flash "#{otp_setup_notice_flash}, please make note of your recovery codes"
      return_response add_recovery_codes_view
    end
  end
end
```

If we wanted, we could even change how recovery codes are generated. For example, we could use UUIDs.

```rb
# app/misc/rodauth_main.rb
class RodauthMain < Rodauth::Rails::Auth
  configure do
    # ...
    new_recovery_code { SecureRandom.uuid }
  end
end
```

Now, when we disable TOTP, we can see that the recovery codes have been deleted as well.

```sql
DELETE FROM "account_recovery_codes" WHERE ("id" = 1)
```

And when we setup TOTP again, we're now shown the generated recovery codes straight away, and there is our flash message. We also see that the recovery codes are now UUIDs.

## Downloadable recovery codes

Let's now improve the design of this page. We'll start by importing view templates for recovery codes:

```sh
$ rails generate rodauth:views recovery_codes
# create  app/views/rodauth/recovery_codes.html.erb
# create  app/views/rodauth/add_recovery_codes.html.erb
# create  app/views/rodauth/recovery_auth.html.erb
# ...
```

In the `add_recovery_codes.html.erb` template, I'll just paste in some ERB code:

```erb
<!-- app/views/rodauth/add_recovery_codes.html.erb -->
<% if rodauth.recovery_codes.any? %>
  <p class="my-3">
    Copy these recovery codes to a safe location.
    You can also download them <%= link_to "here", "data:,#{rodauth.recovery_codes.join("\n")}", download: "myapp-recovery-codes.txt" %>.
  </p>

  <div class="d-inline-block mb-3 border border-primary rounded px-3 py-2">
    <% rodauth.recovery_codes.each_slice(2) do |code1, code2| %>
      <div class="row text-primary text-left">
        <div class="col-lg my-1 font-monospace"><%= code1 %></div>
        <div class="col-lg my-1 font-monospace"><%= code2 %></div>
      </div>
    <% end %>
  </div>
<% end %>
<!-- ... -->
```

We'll now be displaying the recovery codes in two columns. We've also added a download link. We could've created a backend endpoint for this, but then we'd need to password protect it, to maintain the same level of security. Instead, we've embedded the recovery codes in a data URI, and coupled that with the `download` attribute.

Now, when we setup TOTP again, the recovery codes page is looking much nicer. And we can even download them.

## Closing words

With relatively few changes, we were able to setup recovery codes as a backup multifactor authentication method. Rodauth provided a lot of the functionality out-of-the-box, we just tweaked the configuration to make the recovery codes more discoverable.

Now users who have multifactor authentication setup are a lot less likely to lock themselves out of their account.
