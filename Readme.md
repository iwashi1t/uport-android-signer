## Uport android signer

This library is used to create and manage keys for uport account. 
It supports creating keyPairs from seed phrases,
protecting these keys with authenticated encryption (android lock-screen / fingerprint),
signing ETH transactions and signing uPort specific JWTs.

Where available, keys and seeds created by this lib will be protected by 
encryption backed by ARM Trusted Execution Environment (TEE).

Note: The curve used for ETH signing is not backed by the TEE,
therefore private keys exist in memory while in use but are encrypted with TEE keys at rest.

### Import

in your main `build.gradle`:
```groovy

allprojects {
    repositories {
        //...
        maven { url 'https://jitpack.io' }
    }
}
```
in your app `build.gradle`:
```groovy
uport_signer_version = "v0.2.2"
dependencies {
    //...
    implementation "com.github.uport-project.uport-android-sdk:signer:$uport_signer_version"
}
```

### Usage

#### Key protection

When creating or importing keys/seeds, you must specify the protection `KeyProtection.Level` desired.
When using such a key/seed, the protection level is enforced based on the option used during creation.

Keys and seeds are always protected with hardware backed keys, where available.
This option is for choosing user authentication requirements. 

The options are:
* SIMPLE - usable without user authentication
* PROMPT - when fingerprint sensor is available and configured will prompt for fingerprint whenever used.
            otherwise it will default to normal android lockscreen using pin/pattern/password
* SINGLE_PROMPT - uses android lockscreen to confirm user authentication
            only if the user has not authenticated in the last 30 seconds.
* CLOUD - reserved/unused - defaults to SIMPLE

> #### Important notes:
> * On KitKat, all `KeyProtection.Level` options default to `SIMPLE`
> * On Lolipop, the 30 second timeout window for `SINGLE_PROMPT` is not enforced by the 
`AndroidKeyStore` API, it is emulated by this library 

#### Create a seed:

This creates a seed that is used for future key derivation and signing:
The seed is representable by a bip39 mnemonic phrase.

```kotlin
UportHDSigner().createHDSeed(activity, KeyProtection.Level.SIMPLE, { err, rootAddress, publicKey ->
                //seed has been created and is accessible using rootAddress 
                // * the handle is `rootAddress`
                // * the corresponding publicKey in base64 is `publicKey`
                // * if there was an error, those are blank and the err object is non null
                
                // To use the seed, refer to it using this `rootAddress` 
            })
```

You can also import bip39 mnemonic phrases:

```kotlin

//bip39 mnemonic phrase:
val phrase = "vessel ladder alter ... glass valve picture"

UportHDSigner().importHDSeed(activity, KeyProtection.Level.SIMPLE, phrase, { err, rootAddress, publicKey ->

                if (err != null) {
                    //handle error
                } else {
                    assertEquals("0x794a...e96ac8", rootAddress)
                    //seed has been imported and 
                    // * the handle is `rootAddress`
                    // * the corresponding publicKey in base64 is `publicKey`
                }
                 
            })
```

#### Signing

You can use this lib to calculate ETH transaction signatures.
Building and encoding transaction objects into `ByteArray`s is not in the scope of this lib.

You can sign transactions using keys derived from a previously imported seed.
To refer to that seed you must use the `rootAddress` from the seed creation/import callback
Based on the `KeyProtection.Level` used during seed import/creation, a prompt may be shown to the user
on the lock-screen / fingerprint dialog.

```kotlin

val rootAddress = "0x123..." //rootAddress received when creating/importing the seed

//bip32 key derivation
val derivationPath = "m/44'/60'/0'/0/0"

//the transaction payload, base64 encoded
val txPayloadB64 = Base64.encodeToString( transaction.rlpEncode(), Base64.DEFAULT )

//gets shown to the user on fingerprint dialog or on lockscreen, based on `KeyProtection.Level` used
val prompt = "Please sign this transaction"

UportHDSigner().signTransaction(activity, rootAddress, derivationPath, txPayloadB64, prompt, { err, sigData ->
    if (err != null) {
        //handle error
    } else {
        //use sigData r,s,v components
    }
})

```

Note: The requirement to encode the transaction payload as a base64 string is subject to change in future releases

##### uPort specific JWTs

This lib can also produce signatures for uPort specific JWTs:
The method signature is the same but the signing method differs.

Also, do note that in the current version of this API,
 `data` is a Base64 encoded string (not a serialized JSON) - this is subject to change in future releases 

```kotlin

UportHDSigner().signJwtBundle(activity, rootAddress, derivationPath, data, prompt, { err, sigData ->
    if (err != null) {
        //handle error
    } else {
        //use sigData r,s,v components
    }
})

```


### Changelog


#### latest
    * *the signer library has moved to [uport-android-sdk](https://github.com/uport-project/uport-android-sdk)*

#### v0.2.2
    * [feature] enable recoverable signature in JOSE encoding (#11)

#### v0.2.1
    * [bugfix] incorrect exception when activity context was needed (#5)
    * [feature] add functionality to delete keys/seeds (#8)

#### v0.2.0
    * lowered minSDK requirements to 19
        All eth keys and seeds are encrypted with RSA provided by the AndroidKeyStore API, but before SDK 23 that API does not enforce user authentication at runtime so the Keyguard requirement is emulated on Lolipop and not possible on KitKat
    * no longer depending on all of kethereum, only using the `crypto` and `bip39` libraries
    * no longer forwarding the kethereum API to integrating modules.
        If you were depending on that, please add a direct dependency to kethereum.
    * also, updated the kethereum dependency to v0.53. If you were depending on an older version, you need to update too
        
#### v0.1.1
    * to make the `v` recovery param available to calling classes, JWT signing produces `SignatureData` instead of a JOSE encoded signature String
    
#### v0.0.1
    * initial release
