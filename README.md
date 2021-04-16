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
    implementation 'com.github.MoolahTech:mutualfunds-android:0.1.4'
}
```
7. Build your app. If everything passes, you are ready to use the SDK!

## Usage (Staging / UAT)

### KYC

1. Call `KycActivity` with an intent containing email and phone number, and optionally the users' first and last name. The identity token will also be required in a future update, but for now this is enough:
```kotlin
val intent = Intent(activity, KycActivity::class.java)
intent.putExtra("email", "email@gmail.com")
intent.putExtra("phoneNumber", "+919876543210")
intent.putExtra("firstName", "Name")
intent.putExtra("lastName", "Name")
// intent.putExtra("identityToken", "USER_IDENTITY_TOKEN")
this.startActivity(intent)
```
2. Get the KYC result:
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
