Add a feature where an api key token is created on seller onboarding activate (by admin action).
This token will be used and mapped to the seller on the /submitOrder external endpoint + mockup
Currently all tokens use the same encryption key. We need to differentiate between the key used for auth tokens and the key used for seller api keys