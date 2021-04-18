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

## Usage (Staging / UAT)

### KYC

1. Call `KycActivity` with an intent containing email and phone number, and optionally the users' first and last name. The identity token will also be required in a future update, but for now this is enough:
```kotlin
import `in`.savvyapp.mutualfunds_android.kyc.KycActivity

val intent = Intent(activity, KycActivity::class.java)
intent.putExtra("email", "email@gmail.com")
intent.putExtra("phoneNumber", "+919876543210")
intent.putExtra("firstName", "Name")
intent.putExtra("lastName", "Name")
// intent.putExtra("identityToken", "USER_IDENTITY_TOKEN")
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
