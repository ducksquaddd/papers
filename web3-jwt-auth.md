# Bridging Web2 Simplicity and Web3 Security: A JWT-Based Auth Specification

- Author: Crimson M 
- License: CC0 Public domain
- January 13th, 2025

# 1: Introduction
This specification is an outline for how secure authentication can be created using public key and private key cryptography while maintaining the simplicity of web2 auth. Traditional solutions might include someone simply connecting a wallet and the application simply providing all the relevant data, public, or private. This might sound dandy until you realize there is no security measure stopping the user from running a modified extension meant to trick your app (Which might not be an issue in many cases like dApps). With that noted, this system will not work (at least securely, you can do it anyways) for applications that arent completely decentralized, and still host some information off-chain (e.g. social applications, token-gated sites, etc). So this spec attempts to fix these issues without having users jump through any hoops while maintaining web2 *like* auth.

## 1.5: Exigence
I have seen many sites in recent years that *try* to implement a system of blocking access if your wallet/address doesn't meet X criteria after only connecting. This obviously doesn't work if a user is acting in bad faith and is trying to get around these guards.

# 2 Motivation / Why? (To build on above)
I believe modern crypto wallets secured by a private key or mnemonic is the most SIMPLE and secure way of auth, in comparison to an email and password.
 - **Enhanced Security** - By adding an extra verification layer you can **actually** verify wallet ownership.
 - **Improved UX & DX** - This solution provides native JWT auth so it's compatible with existing username/password auth systems, and is compatible with virtually every single wallet. 
 - **Flexibility** - This system is useful for virtually any use case, and is at its base just an auth system.

# 3. Requirements 
This implementation does have SOME requirements, though minimal and widely implemented.
- The blockchain/wallet must implement signing text/arbitrary data in some form, but this implementation is pretty bare-bones and can be implemented in many ways, you might just risk some UX.
- Secure endpoint with some JWT auth system

# 4. Implementation
Finally, the boring part is over onto the implementation! Keep in mind this system is very simple and can be messed with if you need something more custom.

## 4.1 Challenge implementation
- Generate a unique cryptographic challenge 
- This function that generates a challenge should take in an address
- For improved UX and security, the challenge should append/prepend a string along the lines of "This is a unique challenge to verify that you own this wallet. If this is being sent to you do NOT sign it". To stop a blind-signing attack
- You will probably want to include an expiration time for this challenge.
- Finally, you will likely want to host this functionality over a POST endpoint, so clients can receive the challenge

This can all be done using JWT's, here's a Javascript example
```js
function createChallenge(address) {
    const randomChallenge = crypto.randomBytes(32).toString("hex");
    const advisoryMessage = "This is a unique challenge to verify wallet ownership at https://example.com, and is valid for 10 minutes. Do NOT sign this if it's unexpected.";
    const challengeString = `${randomChallenge}\n\n${advisoryMessage}`;
    
    return jwt.sign({ address, challenge: challengeString }, JWT_SECRET, { expiresIn: "10m" });
}
```

## 4.2 Challenge Verification
- Accept the signed challenge, wallet address, and public key as input.
- Decode the JWT to extract the original challenge.
- Verify the JWT is valid (Hasnt expired, etc)
Finally, if the JWT itself is valid, ensure the signature is valid and the publicKey correlates to the address if that's not part of the verifying process.
- If valid, generate a new authentication JWT containing user details and any required metadata.

Javascript Example
```js
function verifyChallenge(token, signature, publicKey, address) {
    const decoded = jwt.verify(tokens, JWT_SECRET);
    const isValid = verifySignature(decoded.challenge, signature, publicKey, address);
    
    if (!isValid) throw new Error("Invalid signature");

    return jwt.sign(
        { address, signedChallenge: signature, metadata: "any additional info" },
        JWT_SECRET,
        { expiresIn: "100d" }
    );
}
```

This is the end of the server-side spec, everything beyond this point is just normal JWT management (Being able to decode the real auth JWT and fetch metadata, etc)

## 4.3 Client-side implementation 
- First, the client will want to fetch a challenge from the previously created endpoint. 

Javascript example
```js
async function requestChallenge(address) {
    const response = await fetch("/api/create", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ address }),
    });

    if (!response.ok) {
        throw new Error("Failed to request challenge");
    }

    return response.json(); // Returns { challenge, JWT }
}
```

- After fetching the challenge the client will sign the challenge string using their respective wallet.
- walletObject - walletObject being the window.wallet instance with the functions provided
- challenge - the challenge just being the challenge string

Javascript example
```js
async function signChallenge(walletObject, challenge) {
    try {
        const signature = await wallet.signMessage(challenge);
        return signature;
    } catch (error) {
        throw new Error("Failed to sign challenge: " + error.message);
    }
}
```

Finally, the client will send this signed challenge to the server to verify who owns the wallet and receives their auth JWT.

Here is another Javascript example
- Challenge - Challenge being the original challenge sent by the server
- Signature - Signature being the signed challenge
- PublicKey - PublicKey is the wallet public key so that the server can verify the signature
- address - address simply being the connected wallet address

```js
async function verifyChallenge(challenge, signature, publicKey, address) {
    const response = await fetch("/api/verifyChallenge", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ challenge, signature, publicKey, address }),
    });

    if (!response.ok) {
        throw new Error("Challenge verification failed");
    }

    return response.json(); // Returns { JWT, address }
}
```

Now finally, we're at the point where you can do whatever you'd normally do as now this JWT system is the same as any other ordinary JWT setup. But for reference here are some bullet points

- Store the JWT somewhere secure in the client
- Implement a system on the server to protect routes by this JWT, just verify the JWT is a valid session JWT not a challenge JWT.

## Implementations
- [AddrAuth](https://github.com/ducksquaddd/AddrAuth) - By yours truly, and what inspired this paper.
