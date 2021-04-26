# Documentation on how to use the Savvy Android SDK

### The following will walk you through the installation steps

1. Get your github username added to our Jitpack account. *This is required to access the build artifacts (aar / jar files).*

2. Sign up for your own Jitpack account. (https://jitpack.io/).

3. Generate an authentication token for yourself on Jitpack.

4. Add the token to $HOME/.gradle/gradle.properties: `authToken=AUTHENTICATION_TOKEN`

5. Then use authToken as the username in your build.gradle: 
```gradle
repositories {
        maven {
            url "https://jitpack.io"
            credentials { username authToken }
        }
 }
 ```
6. Install the SDK in your android app as you would regularly:
```gradle
dependencies {
    implementation 'com.github.MoolahTech:mutualfunds-android:1.0.0-beta2'
}
```
7. Build your app. If everything passes, you are ready to use the SDK!

To use the SDKs, you have to first get access to your access key and secret key. Request your SPOC for these credentials. Staging credentials will be given first, and then production credentials will be issued once a round of testing has been performed. The API urls will also be shared via your SPOC.
**The secret key must be kept secret**. Please make sure this key is not on the phone, or anywhere in your database or permanent storage. It must be kept in your live environment as a env. variable or use other similar key storage mechanisms like AWS KMS. Leakage of your secret key could compromise all your users and could lead to very bad things.

## User creation

Every request from the SDK to the our API must be authenticated using your access key (which identifies the partner making the request) and an IDENTITY_TOKEN (which identifies the user making the request). A user must be created via a server-to-server call using your access and secret key. In return, an expiring token is passed back. Please store this token SAFELY, preferably in the android keystore or other similar storage mechanism and pass it to the SDK.

**User create call**

**POST** ($BASE_URL)**/partners/users**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-PARTNER-SECRET-KEY | string | xyzab |

Params (root key must be `user`: `user: { phone_number.... }`):
| name | type | example |
| ---- | ---- |:-------:|
| phone_number | string | 9876543210 OR +919876543210 |
| email | string | test@example.com |
| first_name | string | Foo |
| last_name | string | Bar |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-USER-IDENTITY-TOKEN | string | abcd.efg.hijk |
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-hijk-xyz |

The identity token expires every 24 hours, so please make sure to have a refresh mechanism built in.

**Token refresh call**

**POST** ($BASE_URL)**/partners/users/new_token**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-PARTNER-SECRET-KEY | string | xyzab |

Params (root key must be `user`: `user: { uuid.... }`):
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-rf-rrrr |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-USER-IDENTITY-TOKEN | string | abcd.efg.hijk |
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-hijk-xyz |

## Usage (Staging / UAT)

### KYC

1. Call `KycActivity` with an intent containing email and phone number, and optionally the users' first and last name. The identity token will also be required in a future update, but for now this is enough:
```kotlin
import `in`.savvyapp.mutualfunds_android.kyc.KycActivity

val intent = Intent(activity, KycActivity::class.java)
intent.putExtra("identityToken", "IDENTITY_TOKEN")
intent.putExtra("accessKey", "PARTNER_ACCESS_KEY")
this.startActivity(intent)
```
**Remember, the OTP in the UAT / Staging environment is 123456.**

2. The KYC process consists of:
* Phone number verification using OTP.
* PAN number check. If the PAN is already KYC-verified, then short KYC is triggered, otherwise long KYC is triggered.

**Short KYC:** In addition to the previous steps, the SDK also performs:
* Bank account verification
* Capture of KYC details like date of birth, occupation and PEP check.
* Folio creation

**Long KYC:** This is an involved process, and the SDK performs:
* Capture of KYC details like date of birth, occupation, marital status etc
* PAN card image capture + details submission
* Address proof image capture (aadhaar, passport, drivers' license or voter ID) + details submission
* Signature on plain white paper image capture
* Video verification
* Bank account verification
* Contract signing using aadhaar e-sign
* Submission to KRA for verification

3. Get the KYC result:
```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        ...
}
```
**KYC success**: The result code is Activity.RESULT_OK. The user may proceed to purchase mutual funds.

**KYC failure**: The result code is Activity.RESULT_CANCELED. There was an error during KYC, and user may not proceed. Retrying the KYC will usually *not* work. For recoverable errors, the SDK has a built-in retry mechanism. For these kinds of errors, we recommend logging them and reporting them along with the intent parameters. The intent will contain:
* `errorCode`
* `message`

### Purchase

The purchase process consists of picking which mutual fund the purchase should go into, entering the purchase amount & payment method, and then executing the transaction using UPI or internet banking. The SDK provides the option of both a UI based journey, and also a programmatic journey, where you can create your own UI and pass in the fund into which you would like the user to save. While you can submit the the purchase programmatically, the transaction completion which involves interacting with the payment gateway must be handled by the SDK due to compliance restrictions.
The Purchase SDK assumes that KYC has already been completed on **the same device**.

**UI based journey**
1. Call `PurchaseActivity` with the following parameters:

* IDENTITY_TOKEN (mandatory)
* PARTNER_ACCESS_KEY (mandatory)
* productCode (optional). The fund in which you'd like to invest. If not entered, a screen will be shown for ths user to select. The following product codes are currently supported:

| Product name  | productCode |
| ------------- |:-------------:|
| ICICI Prudential Bluechip Fund - Growth | 1191 |
| ICICI Prudential Multicap Fund - Growth | 121 |
| ICICI Prudential Balanced Advantage Fund - Growth | EDWRG |
| ICICI Prudential Regular Savings Fund - Growth | IMPG |
| ICICI Prudential Gilt Fund - Growth | 53 |
| ICICI Prudential Banking and PSU Debt Fund - Growth | 1587 |
| ICICI Prudential Savings Fund - Growth | 1525 |
| ICICI Prudential Liquid Fund - Growth | 1565 |
| ICICI Prudential Long Term Equity Fund (Tax Saving) - Growth | 01 |
| ICICI Prudential Asset Allocator Fund (FOF) - Growth | AMP |
| ICICI Prudential Regular Gold Savings Fund (FOF) - Growth | 1815 |

* amount (optional). The amount you'd like the invest. If not entered, a text input field will be shown for the user to select.
* preferredPaymentMethod (optional). The payment method that you'd like to use. The following are supported:

| Description | preferredPaymentMethod |
| ----------- |:----------------------:|
| UPI         | UPI                    |
| Internet banking | I                 |

* upiVpa (required if payment method is UPI). If payment method is UPI, you must also pass in the Virtual Payment Address of the user.

### Customization
**Colors: ** To customize the colors, please override the theme in your activity. This is an experimental feature and may have bugs!! We are actively working on fixing this.
