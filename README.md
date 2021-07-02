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

**IMPORTANT: The steps below also detail how to use the SDK to build your own UI. You can only build your own UI if you are a licensed distributor with a valid ARN number.** If you are not a distributor, you must use the standard SDK UI.
In the below documentation, we will use the `<YourUi>` symbol to denote how to build your own UI for the respective step.

## User creation

Every request from the SDK to the our API must be authenticated using your access key (which identifies the partner making the request) and an IDENTITY_TOKEN (which identifies the user making the request). A user must be created via a server-to-server call using your access and secret key. In return, an expiring token is passed back. Please store this token SAFELY, preferably in the android keystore or other similar storage mechanism and pass it to the SDK.

**User create call**

**POST** ($BASE_URL)**/partners/users**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-PARTNER-SECRET-KEY | string | xyzab |

Params (root key must be user: `user: { phone_number.... }`):
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

Params (root key must be user: `user: { uuid.... }`):
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

### Initialization

Initialize the SDK by calling `init`. Make sure to do this before calling any of the activities, or before using the SKD in any way:
```kotlin
MFSDK.init(Context, PARTNER_ACCESS_KEY, USER_IDENTITY_TOKEN, isProduction: Boolean)
```

### KYC

1. Call `KycActivity`:
```kotlin
import `in`.savvyapp.mutualfunds_android.kyc.KycActivity

val intent = Intent(activity, KycActivity::class.java)
intent.putExtra("phoneNumber", "9898989898")
this.startActivity(intent)
```

2. The KYC process consists of:
* PAN number check. If the PAN is already KYC-verified, then short KYC is triggered, otherwise long KYC is triggered.

`<YourUI>` 

Pan check consists of 2 steps:
Check Pan:
```kotlin
val req = CheckPanRequest(panNumber: String, context: Context)
req.call(view: View, loader: Loadable?, callback: (response: CheckPanResponse?) -> Unit, failureCallback: ((response: CheckPanResponse?) -> Unit)?

// view: Root view of current screen, so that system error messages may be displayed (Mandatory)
// loader: Implementation of Loadable interface. This is used if you want to change state of a particular view to show a loading icon (Optional)
// callback: A handler for a successful response. Returns a CheckPanResponse (Mandatory)
// failureCallback: A handler for an unsuccessful response (Optional)
```
CheckPanResponse contains the method `isShortKyc` which must be called. If true, you can proceed to SubmitPanRequest. Otherwise redirect to the Long KYC process which is a separate SDK. **THIS IS VERY IMPORTANT**. Not doing this is a compliance violation, and can result in action.

Submit Pan:
```kotlin
val req = SubmitPanRequest(context: Context)
req.call(view: View, loader: Loadable?, callback: (response: CheckPanResponse?) -> Unit, failureCallback: ((response: CheckPanResponse?) -> Unit)?
```
If submit PAN is successful, you can go to the next step

`</YourUI>`

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

### Mutli-folio functionality

Once the user has finished their KYC, mutiple folios may be created for the same user if the partners' use-case demands it.
This is a raw API call. On successful creation of the folio, a callback will be sent to the partner. The `partner_transaction_id` used to create the folio can be used in subsequent calls for transactions.

**POST** ($BASE_URL)**/partners/icici_pru/folios/create_additional**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-USER-IDENTITY-TOKEN | string | xyzab |

Params (root key must be deposit: `deposit: { amount.... }`):
| name | type | example |
| ---- | ---- |:-------:|
| partner_transaction_id | string, generated by you | XYZ123 |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|

The status (success / failure) will be received as regular from the callbacks.

### Purchase

The purchase process consists of picking which mutual fund the purchase should go into, entering the purchase amount & payment method, and then executing the transaction using UPI or internet banking. The SDK provides the option of both a UI based journey, and also a programmatic journey, where you can create your own UI and pass in the fund into which you would like the user to save. While you can submit the the purchase programmatically, the transaction completion which involves interacting with the payment gateway must be handled by the SDK due to compliance restrictions.

There are 2 kinds of purchase transactions, under which there are 2 types of payment gateways you can be used:

### Regular / one-time purchase

Call `PurchaseActivity` with the following parameters in the intent:

* **pgType: PGType converted to String** (required) This can either be PGType.INTERNAL or PGType.EXTERNAL. Internal is the AMCs payment gateway, and external is the Savvy payment gateway. In most cases, you want to use the INTERNAL PG, but the AMC payment gateways are sometimes not functional. As a backup, the partner may choose to use the EXTERNAL PG to minimize user disruption. To pass this in, please do `PGType.INTERNAL.name`
* **amount: Int?** (optional for INTERNAL PG, mandatory for EXTERNAL PG)
* **partnerTransactionId: String** (required) Partner generated ID. Use a uuid or similar globally unique ID. This will be used to update your backend of transaction events.
* **phoneNumber: String** (required)
* **productCode: String** (required) See below for supported product codes
* **preferredPaymentMethod: String** (optional, and only used for INTERNAL PG). The payment method that you'd like to use. See below for supported methods.
* **upiVpa: String** (required if payment method is UPI, and only used for INTERNAL PG) If payment method is UPI, you must also pass in the Virtual Payment Address of the user.
* **folioId: String** (required if using multi-folio functionality)

Example:
```kotlin
val intent = Intent(activity, PurchaseActivity::class.java)
intent.putExtra("productCode", "1565")
intent.putExtra("amount", 100)
intent.putExtra("partnerTransactionId", System.currentTimeMillis().toString())
intent.putExtra("phoneNumber", "9898989898")
intent.putExtra("pgType", PGType.INTERNAL.name)

this.startActivityForResult(intent, 1)
```

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

| Description | preferredPaymentMethod |
| ----------- |:----------------------:|
| UPI         | UPI                    |
| Internet banking | I                 |

`PurchaseActivity` response:
```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        ...
}
```

In case of success, the resultCode will be Activity.RESULT_OK, in case of failure it will be Activity.RESULT_CANCELLED.
The intent contains the key `message`, which you may choose to show to the user.

### Collect request

There could be a use-case where you need to trigger a payment request programatically from your backend. This is primarily used for out-of-band and out-of-app payment requests. In these cases, you can interact directly with the API to achieve this functionality:

In case of UPI, this will automatically trigger a payment request, and the user will receive a notification from their payment app such GPay, PhonePe etc.

**POST** ($BASE_URL)**/partners/deposits**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-USER-IDENTITY-TOKEN | string | xyzab |

Params (root key must be deposit: `deposit: { amount.... }`):
| name | type | example |
| ---- | ---- |:-------:|
| partner_transaction_id | string, generated by you | XYZ123 |
| amount | int | 100 |
| product_code | string | 1565 |
| preferred_payment_method | string | `UPI` for upi, `I` for net banking, `UPI_MANDATE` for upi mandate trigger, `API_MANDATE` for api mandate trigger |
| upi_vpa | string | abc@okhdfcbank.com |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| gateway_url | string | redirect to this URL in case of net banking (only used for `I` |
| message | string | Message from the UPI app (gpay, phonepe etc) |
| deposit | object | { id: 'abc', uuid: 'xyz' } |
| folio | object | { { id: 'abc', uuid: 'xyz' } |

The status (success / failure) will be received as regular from the callbacks.

### Mandate / Standing Instruction

Make a **`MandatePurchaseActivity`** activity with the following params in the intent:

* **mandateType: String** (required) One of ["upi". "api"]
* **maxOngoingAmount: Double** (required) This is the maximum that you can debit from the user everyday. This must match the minimum required for the mutual fund.
* **partnerTransactionId: String** (required)
* **productCode: String** (required)
* **startDate: String** (required) The starting date of the mandate. Format mentioned in the example below.
* **endDate: String** (required) The end date of the mandate. We recommend keeping this far in the future to avoid re-registering mandates. The max is 99 years.
* **upiVpa: String** (optional) Please DON'T pass this in for staging as UPI VPA checking is unavailable on staging. Pass this in on production to pre-fill the VPA.
* **frequency: String** (optional with default ADHOC) One of: [ 'ONETIME', 'ADHOC', 'INTRADAY', 'DAILY', 'WEEKLY', 'MONTHLY', 'QUARTERLY', 'SEMIANNUALLY', 'YEARLY' ]

Example:
```kotlin
val sdf = SimpleDateFormat("yyyy-MM-dd", Locale.UK)
val cal = Calendar.getInstance()
cal.add(Calendar.YEAR, 20)
val startDateString = sdf.format(Date())
val endDateString = sdf.format(cal.time)

val intent = Intent(activity, MandatePurchaseActivity::class.java)
intent.putExtra("mandateType", "api")
intent.putExtra("productCode", "1565")
intent.putExtra("maxOngoingAmount", 1000.toDouble())
intent.putExtra("partnerTransactionId", System.currentTimeMillis().toString())
intent.putExtra("startDate", startDateString)
intent.putExtra("endDate", endDateString)
intent.putExtra("frequency", "DAILY")

this.startActivityForResult(intent, 2)
```

### Redemption
The redemption process consists of picking which mutual fund the redemption should come from and then executing the transaction. The SDK provides the option of both a UI based journey, and also a programmatic journey, where you can create your own UI and pass in the fund from which you would like the user to withdraw. While you can submit the the redemption programmatically, we recommend using the SDK due to the complexity of the process involved.

For liquid funds (code 1565), there is an option of instant withdrawal. Upto to 90% of balance or Rs 50,000 (whichever is lower) can be transferred immediately via IMPS and lands in the users' bank account within 15 seconds. The rest of the money is credited within 1 business day. The instant option is not available for non-liquid funds.

All redemptions (excluding instant) are subject to OTP approval.

**UI based journey**
1. Call `RedemptionActivity` with the following parameters:

* **productCode (mandatory)** The fund from which the redemption should happen. Refer to above table for the product codes.
* **partnerTransactionIdInstant: String** (optional) Partner generated ID. Use a uuid or similar globally unique ID. This will be used to update your backend of transaction events. This will be used for instant withdrawal transactions. This is valid only for liquid funds.
* **partnerTransactionIdRegular: String** (required) Partner generated ID. Use a uuid or similar globally unique ID. This will be used to update your backend of transaction events. This will be used for regular (OTP-based) withdrawal transactions. Valid for liquid and other funds.
* * **folioId: String** (required if using multi-folio functionality)

```kotlin
val intent = Intent(activity, RedemptionActivity::class.java)
intent.putExtra("productCode", "1565")
intent.putExtra("partnerTransactionId", "1234")
this.startActivity(intent)
```

2. Get the redemption result:
```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        ...
}
```
**Redemption success**: The result code is Activity.RESULT_OK. The balance will be updated.

### Balance Check

The balance check SDK is built as a fragment that can be included anywhere. Simply create a new fragment: `BalanceFragment()` and display wherever you'd like. For example:
```kotlin
supportFragmentManager.beginTransaction().replace(R.id.content_main, BalanceFragment.newInstance()).commit()
```

If using the multi-folio functionality, pass in the folioId as well:
```kotlin
supportFragmentManager.beginTransaction().replace(R.id.content_main, BalanceFragment.newInstance()).commit()
```

Alternatively, to fetch the balance via API:
**GET** ($BASE_URL)**/partners/users/mf_balance**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-USER-IDENTITY-TOKEN | string | xyzab |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| available_investment_balance | Double | 1000.12 |
| latest_deposit_date | Date | 01/01/2021 |
| investment_plus_pending_balance | Double | 1500.12, includes money not invested yet |

### Callbacks

There are a few asynchronous events that you will need updates about. Currently, there are 4 events:

1. User long KYC status update
2. Deposit transaction status update
3. Withdrawal transaction status update
4. Mandate registration status update

While the params sent for each of these callbacks is different, each callback is sent with a hash string. This hash string is a pipe-joined string of all the params sent, which is then hashed using HMAC with SHA256 using your secret key. You **must** verify this hash at your end, otherwise attackers might simply be able to spoof requests to your open endpoints.

**User long KYC status update**
Params sent:
```ruby
      {
        transaction_type: 'long_kyc',
        user_id: <UUID>,
        status_is: <CURRENT STATUS>,
        status_was: <PREVIOUS STATUS>,
        hash: <HASH STRING>
      }
```
The possible statuses are: `'pending', 'accepted', 'rejected'`
The hash string is generated as follows:
```ruby
hash_string = "transaction_type|uuid|status_is|status_was"
hash = HMAC('sha256', hash_string, secret_key)
```

**User folio opening status update**
Params sent:
```ruby
      {
        transaction_type: 'folio_open',
        uuid: <UUID>,
        status_is: <CURRENT STATUS>,
        status_was: <PREVIOUS STATUS>,
        hash: <HASH STRING>
      }
```
The possible statuses are: `'true', 'nil'` 
The hash string is generated as follows:
```ruby
hash_string = "transaction_type|uuid|status_is|status_was"
hash = HMAC('sha256', hash_string, secret_key)
```

**Folio opening update**

POSTed to your user callback URL

Params sent:
```ruby
      {
        transaction_type: 'folio_open',
        transaction_id: <YOUR_ID>,
        has_folio_opened: true / false
      }
```

The hash string is generated as follows:
```ruby
hash_string = "transaction_type|transaction_id|has_folio_opened"
hash = HMAC('sha256', hash_string, secret_key)
```

**Deposit transaction status update**
Params sent:
```ruby
      {
        transaction_type: 'deposit',
        transaction_id: <YOUR_ID>,
        status_is: <CURRENT STATUS>,
        status_was: <PREVIOUS STATUS>,
        amount: <AMOUNT>
        hash: <HASH STRING>
      }
```
The possible statuses are: `'in_progress', 'pending_investment', 'completed', 'error'`

* in_progress is when the deposit is initiated by the partner
* pending_investment is when the money is debited from the users' bank account
* completed is when the money has been invested and units are allocated
* error is when the transaction has failed

The hash string is generated as follows:
```ruby
hash_string = "transaction_type|transaction_id|status_is|status_was|amount"
hash = HMAC('sha256', hash_string, secret_key)
```

**Withdrawal transaction status update**
Params sent:
```ruby
      {
        transaction_type: 'withdrawal',
        transaction_id: <YOUR_ID>,
        status_is: <CURRENT STATUS>,
        status_was: <PREVIOUS STATUS>,
        amount: <AMOUNT>
        hash: <HASH STRING>
      }
```
The possible statuses are: `'pending', 'completed', 'error'`

* pending is when the withdrawal has been initiated
* completed is when the money has been returned to the users' bank account
* error is when the transaction has failed

The hash string is generated as follows:
```ruby
hash_string = "transaction_type|transaction_id|status_is|status_was|amount"
hash = HMAC('sha256', hash_string, secret_key)
```

**Mandate registration status update**
Params sent:
```ruby
      {
        transaction_type: 'mandate',
        transaction_id: <YOUR_ID>, # This is the ID of the first transaction of the mandate. This will also be the ID of a deposit.
        status_is: <CURRENT STATUS>,
        status_was: <PREVIOUS STATUS>,
        hash: <HASH STRING>
      }
```
The possible statuses are: `'pending', 'accepted_by_user', 'completed', 'error', 'revoked'`

* pending is when the mandate process has been initiated
* pending_approval is when the user has given the appropriate permissions, and is pending verification with the bank
* completed is when the mandate is accepted by the bank
* error is when the mandate is cancelled for some reason (wrong permission, bank rejection etc)

The hash string is generated as follows:
```ruby
hash_string = "transaction_type|transaction_id|status_is|status_was|amount"
hash = HMAC('sha256', hash_string, secret_key)
```

### Customization
**Colors: ** To customize the colors, please override the following colors:

```xml
<color name="savvy_color_primary">#005B75</color>
<color name="savvy_color_secondary">#FF03DAC5</color>
<color name="savvy_color_accent">#EF9F39</color>
<color name="savvy_color_on_primary">#FFF</color>
<color name="savvy_color_on_secondary">#FF03DAC5</color>
<color name="savvy_color_screen_background">#FFF</color>
<color name="savvy_color_on_background">#FF757575</color>
<color name="savvy_stepper_default">#D8D8D8</color>
```
