Add a feature where an api key token is created on seller onboarding activation (by admin action).
This token will be used and mapped to the seller on the /submitOrder external endpoint + mockup (for the mockup, modify the endpoint "getmainSellers" that returns the sellers list + token to return the api key token to the same endpoint without any changes)
Currently all tokens use the same encryption key. We need to differentiate between the key used for auth tokens and the key used for seller api keys

- Generate API key for the seller on activation
- Map the api key to seller id
- Make sure the API key uses a different encryption key than the portal login
- Save the generated tokens in the database (encrypted)
- Add an entry in the seller portal for the seller to view his API key
- Add a button for the user to rotate/reset the api key (postponed)
- Allow api key rotation from the seller side (postponed)
