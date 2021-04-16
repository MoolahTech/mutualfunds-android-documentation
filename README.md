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

