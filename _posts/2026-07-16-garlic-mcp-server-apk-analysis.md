## APK Analysis via Garlic MCP Server

I've been building [garlic](https://github.com/neocanable/garlic) as a decompiler for Android DEX/APK files. Recently I wrapped it as an MCP (Model Context Protocol) server, which means LLMs like Claude can directly invoke garlic's tools to decompile and analyze APKs. This turned out to be surprisingly effective, so I want to walk through how it works using a real example: Tinder v17.24.0.

----

#### 1. What is an MCP Server for Reverse Engineering

MCP is a protocol that lets LLMs call external tools. Instead of asking "what's in this APK?" and having the model guess from training data, you give it tools: decompile a class, read the manifest, query the call graph. The model drives the analysis itself, calling tools as needed.

Garlic's MCP server exposes these tools:

- `analyze` -- one-shot: decompile + call graph + DuckDB import
- `decompile` -- decompile APK/DEX/JAR to Java source
- `android_manifest` -- read the binary AndroidManifest.xml
- `call_graph` -- generate call graph from DEX/APK
- `cg_import` -- import call graph into DuckDB
- `cg_query` -- run SQL queries on the call graph
- `dump_info` -- show class/dex structure (like javap)

The key insight is the **one-shot pipeline**. A single `analyze` call decompiles the APK to Java source, builds a full call graph, and imports everything into DuckDB. Now you have ~100K Java files and a million-edge call graph that you can query with SQL.

----

#### 2. The Problem: XAPK Format

The first challenge with Tinder was the file format. The file I had was 148MB but wasn't a standard APK:

```shell
$ unzip -l tinder.apk
 127326609  com.tinder.apk
  25466271  config.arm64_v8a.apk
    614610  config.en.apk
   2228592  config.mdpi.apk
      2048  manifest.json
      9941  icon.png
```

This is an **XAPK v2** -- a wrapper containing split APKs (Android App Bundle output). The inner `com.tinder.apk` (127MB) is the real APK with all the code. The config APKs hold platform-specific native libs and resources.

The `analyze` MCP tool expects a real APK, so I extracted the inner APK first and ran the pipeline on that.

----

#### 3. Running the Analysis Pipeline

The one-shot command is simple:

```
analyze(path="/path/to/com.tinder.apk", output_dir="./tinder_analysis")
```

Behind the scenes, garlic:
1. Enumerates all DEX files in the APK
2. Decompiles every class to Java source
3. Builds a complete call graph (nodes = methods, edges = call sites)
4. Imports everything into a DuckDB database

The result:

```shell
$ ls tinder_analysis/
analysis.duckdb   cg/               decompiled/

$ find decompiled/com/tinder -name "*.java" | wc -l
   55706

$ find decompiled -name "*.java" | wc -l
   98603
```

98,603 Java files. 55,706 of them are Tinder's own code (`com.tinder.*`). The rest are third-party libraries: AndroidX, Jetpack Compose, FaceTec, Firebase, Sentry, OkHttp -- everything decompiled to readable source.

----

#### 4. The Call Graph in DuckDB

The most useful part is the SQL-queryable call graph. Let me show some queries.

The database has two main tables:

```sql
DESCRIBE java_cg_nodes;
-- node_id BIGINT, method_raw VARCHAR, node_type BIGINT, api_type BIGINT

DESCRIBE java_cg_edges;
-- src_id BIGINT, dst_id BIGINT
```

**Total size:**

```sql
SELECT COUNT(*) as nodes, (SELECT COUNT(*) FROM java_cg_edges) as edges FROM java_cg_nodes;
```

| nodes | edges |
|-------|-------|
| 586,073 | 1,491,572 |

586K methods, 1.49M call edges. That's a lot of graph.

**Package distribution:**

```sql
SELECT 
  CASE 
    WHEN method_raw LIKE 'Lcom/tinder/%' THEN 'com.tinder'
    WHEN method_raw LIKE 'Landroidx/%' THEN 'androidx'
    WHEN method_raw LIKE 'Lcom/google/%' THEN 'com.google'
    WHEN method_raw LIKE 'Lcom/facebook/%' THEN 'com.facebook'
    WHEN method_raw LIKE 'Lkotlin/%' THEN 'kotlin'
    ELSE 'other'
  END as package,
  COUNT(*) as methods
FROM java_cg_nodes
GROUP BY package
ORDER BY methods DESC;
```

The majority are Tinder's own code (~390K methods), followed by AndroidX, Google libraries, Kotlin runtime, and Facebook SDKs.

**Finding all API endpoints** from string constants is straightforward too:

```sql
SELECT str FROM string_nodes WHERE is_url = true;
```

This returns all URLs found in the entire codebase, including the production API endpoints:

```
https://api.gotinder.com
https://data.gotinder.com
https://imageupload.gotinder.com
https://sharemydate.gotinder.com
https://etl.tindersparks.com
https://api.ue1.s11.tstaging.com
```

The URL classification is done during decompilation -- garlic annotates string constants with their semantic type (URL, UUID, file path, encryption key reference, etc.).

----

#### 5. Reading the Android Manifest

After the pipeline runs, the decompiled manifest is available as a plain XML file:

```shell
$ head -5 tinder_analysis/decompiled/AndroidManifest.xml
<manifest package="com.tinder" android:versionName="17.24.0"
          android:compileSdkVersion="36">
<uses-sdk android:minSdkVersion="32" android:targetSdkVersion="35"/>
```

The MCP server also has a dedicated `android_manifest` tool that returns the parsed manifest with all attributes resolved.

From the manifest, I found some interesting metadata:

```xml
<meta-data android:name="com.bugsnag.android.API_KEY"
           android:value="745b354d173a082bd8bbf7a1501df2e6"/>
<meta-data android:name="io.branch.sdk.BranchKey"
           android:value="key_live_mhkKfK7Br0UHcrNnmV6CVhbosDoGS8JH"/>
<meta-data android:name="com.google.android.geo.API_KEY" .../>
```

Tinder runs **both** Sentry and Bugsnag for crash monitoring (dual coverage). The `firebase_analytics_collection_enabled` is explicitly set to `false`, which is interesting for a data-driven app. And there's a Branch.io key for deep link attribution.

----

#### 6. Diving into the Auth Module

The auth module at `com.tinder.feature.auth` is the most architecturally interesting part of the codebase. It spans ~1,200 files organized as feature modules:

```
feature/auth/
  captcha/          -- Arkose Captcha / HCaptcha challenge
  collect/email/    -- Email address collection
  consent/          -- User consent screens
  email/otp/        -- Email OTP verification
  google/           -- Google Sign-In
  inboundsms/       -- Carrier SMS auto-verification
  internal/
    activity/       -- AuthStartActivity, TermsOfServiceActivity
    authflow/       -- ** Auth flow state machine (CORE) **
    di/             -- Hilt DI modules
    fragment/       -- LoginFragment (login UI)
    phoneverification/ -- Phone verification sub-flow
    viewmodel/      -- LoginViewModel
  passkeys/         -- Google Passkey (passwordless)
  phone/
    number/         -- Phone number input
    otp/            -- Phone OTP verification
  usecases/         -- Interface layer
```

The auth system has three layers: **AuthType** (which auth method), **AuthFlowStep** (state machine), and **AuthStepV2** (step stack).

----

##### 6.1 AuthType -- 9 Authentication Methods

```java
public enum AuthType {
    LINE("line"),
    GOOGLE("google"),
    TINDER_SMS("sms"),        // primary path
    PUSH("push"),
    STACKS("stacks"),
    EMAIL("email"),
    CREATE_ACCOUNT("create_account"),
    LOGIN("login"),
    PASSKEY("passkey");
}
```

Each maps to a backend `key` string sent to the Auth Gateway Service. `TINDER_SMS` is the primary flow for most users.

----

##### 6.2 AuthFlowStep -- The State Machine

The state machine is defined as a Kotlin sealed interface with 10 states:

```
                  +-------------+
                  |  Processing |
                  +------+------+
                         |
              +----------+----------+----------+
              |          |          |          |
      +-------+--+ +-----+------+ +-+--------+ |
      |  Initial  | |   Next     | |  Consent | |
      | StepReady | | StepReady  | | Required | |
      +------+----+ +-----+------+ +----------+ |
             |            |                      |
             |  +---------+----------+           |
             +->|   Authenticated    |<----------+
                +--------+----------+
                         |
              +----------+----------+
              |          |          |
        +-----+---+ +---+----+ +---+----+
        |  Error   | |  Ban   | |Warning |
        +----+-----+ +--------+ +--------+
             |
        +----+------+
        | Cancelled  |
        | AuthOutage |
        +------------+
```

The flow is managed by `AuthFlowViewModel`, which holds a `StateFlow<AuthFlowState>`:

```java
// AuthFlowState fields (reconstructed)
boolean isRecoveringAccount;
AuthType initialAuthType;
List<AuthStepV2> authStepStack;  // step history / back stack
AuthFlowStep authFlowStep;        // current state
```

The **step stack** is the key design: each step gets pushed on as the user progresses. If a step fails (e.g. invalid OTP), the ViewModel pops back to the previous step rather than restarting from scratch.

----

##### 6.3 AuthStepV2 -- The Step Types

There are 10 possible steps in the verification chain:

| Step | Purpose |
|------|---------|
| `CaptchaStep` | Bot challenge (Arkose Labs or HCaptcha) |
| `CollectEmail` | Email address input |
| `EmailOtp` | Email OTP code verification |
| `Onboarding` | New user profile setup |
| `Phone` | Phone number input |
| `PhoneOtp` | SMS OTP code verification |
| `InboundSms` | Carrier SMSC silent verification |
| `IneligibleTinderUEmail` | Tinder U email rejection |
| `AccountRecoveryAwaitEmailMagicLink` | Recovery email link |
| `Passkey` | Google Passkey auth |

----

##### 6.4 AuthRequest -- 27 Request Types

The ViewModel communicates with the backend by submitting `AuthRequest` objects. The sealed interface has 27 variants covering every auth action:

```
Captcha, CollectEmail, ChallengeRecovery,
EmailForAccountRecovery, EmailMagicLinkOtp,
EmailOtp, EmailOtpResend, DismissEmailOtp,
DismissTinderUIneligibleEmail,
Facebook, Google, Line,
Phone, PhoneOtp, PhoneOtpResend, InboundSms,
Refresh, PreEmail,
ExistingPhoneOtp, ExistingPhoneOtpResend,
CreateNewAccount, DismissPasskey,
InitiatePasskey, ValidatePasskey,
InitiateAuthenticatePasskey, AuthenticatePasskey
```

Each request type carries its own payload -- a phone number for `Phone`, an OTP code for `PhoneOtp`, a Google credential for `Google`, etc.

The backend responds with an `AuthResult`:

- `Authenticated` -- login succeeded, contains user token
- `Response` -- needs more steps, contains the next `AuthStepV2`
- `Error` -- various error sub-types (Ban, Warning, IOError, rate limits...)

----

##### 6.5 ViewModel in Action

The `AuthFlowViewModel` constructor injects 16 dependencies. Key ones:

| Injection | Role |
|-----------|------|
| `SubmitAuthRequestV2Impl` | Sends auth requests to AGS backend |
| `SaveInitialAuthTypeImpl` | Remembers auth method across retries |
| `PasskeyAuthFlowDiagnostics` | Passkey-specific logging |
| `FireworksAuthTracker` | Analytics event tracking |
| `SaveBanImpl` | Persists ban state |
| `LoginObserver` | Login lifecycle callbacks |

The ViewModel's key methods (reconstructed from the decompiled state machine):

| Method | Function |
|--------|----------|
| `d0()` | `continueSetInitialAuthStep` -- bootstrap the flow |
| `e0()` | `prePasskeyData` -- check if Passkey is available |
| `f0()` | `applyInitialAuthStep` -- push first step |
| `g0()` | `checkIfAuthIsUp` -- health check before proceeding |
| `h0()` | Parse error string to find correct back-step |
| `i0()` | `handleBan` -- central error handler (15+ error types) |
| `j0()` | `handleAuthenticated` -- save token, transition to main app |
| `k0()` | Find next valid step from available set |
| `v0()` | Update step stack or show error toast |

----

##### 6.6 The Complete Auth Flow

Putting it all together, here is the full authentication sequence reconstructed from the decompiled source:

```
AuthStartActivity
  onCreate() -> check Intent extras (ban, onboarding, challenge state)
  Start LoginFragment (login UI)
    User selects auth method (Google / SMS / Email / Passkey / LINE)
      LoginFragment.Listener.e(AuthType) callback
        Launch AuthFlowActivity

AuthFlowActivity ** Auth Flow Orchestrator **

  1) First entry: d0() -> continueSetInitialAuthStep
     e0() check Passkey availability (via InitiateAuthenticatePasskey request)
     g0() checkIfAuthIsUp() -> health check against AGS
     f0() applyInitialAuthStep() -> push AuthStepV2 onto stack

  2) AuthStepLauncher starts target Activity based on current step:
     CaptchaStep   -> ArkoseChallengeActivity / HCaptchaChallengeActivity
     CollectEmail  -> EmailCollectionActivity
     EmailOtp      -> PhoneNumberOtpActivity (email branch)
     Phone         -> PhoneNumberCollectionActivity  ** primary path **
     PhoneOtp      -> PhoneNumberOtpActivity          ** primary path **
     InboundSms    -> (silent carrier verification)
     Passkey       -> PasskeysFullScreenActivity
     Google        -> AuthGoogleHeadlessActivity
     LINE          -> LineAuthenticationActivity
     Onboarding    -> OnboardingActivity
     IneligibleTinderUEmail -> TinderUIneligibleEmailActivity

  3) Each sub-Activity returns -> ViewModel submits AuthRequest to AGS
     AGS response handling:
       Authenticated -> j0()
         Save auth token
         Track analytics (AuthTokenType.INTERACTIVE or .PUSH)
         Refresh two-factor session
         Update state to Authenticated
         -> close AuthFlow, return to AuthStartActivity
         -> AuthStartActivity.O() -> start MainActivity

       Response(NextStep) ->
         Push next AuthStepV2 onto step stack
         Update state to NextStepReady
         AuthStepLauncher launches next Activity

       Error -> i0() handleBan/Error
         Ban(40301, 40341)    -> save ban record, show BanActivity
         Warning              -> show warning dialog
         InvalidPhoneError    -> show error, back to phone input
         InvalidPhoneOtpError -> show error, retry OTP
         PhoneOtpSendRateLimitError    -> rate limit toast + back
         PhoneOtpInvalidCodeRateLimitError -> rate limit toast
         DenyListedPhoneCarrierError   -> carrier blocked error
         NoExistingAccountForGoogleTokenError -> error code 40076
         DeviceCheckFailureError       -> error code 40098
         InvalidRefreshToken           -> error code 40120
         IOError/UnknownAuthStepError  -> generic error with retry
```

----

##### 6.7 Phone Verification Sub-flow

The SMS verification path has its own dedicated ViewModel (`PhoneVerificationAuthViewModel`) with analytics tracking:

```
PhoneNumberCollectionActivity
  User enters phone number
    SubmitNewPhoneNumberImpl -> submit to AGS
      |
      +-> GoToPhoneOtpStep -> PhoneNumberOtpActivity
      |     User enters OTP code
      |       VerifyUpdatedPhoneOtpImpl -> verify
      |         PhoneOtpSuccess -> complete
      |         Error -> retry with error-specific handling
      |
      +-> InvalidPhoneError -> show error, stay on input
      +-> DenyListedPhoneCarrierError -> carrier blocked
      +-> SeedNewUserError -> route to account creation
```

Each verification attempt fires an `AuthVerifySMSEvent` analytics event with stage markers (`enterPhoneNumber`, `submit`, `enterOTP`) and result codes.

----

#### 7. The AGS Communication Protocol

All auth requests flow through the **Auth Gateway Service** at `api.gotinder.com`. The client adds extensive device fingerprinting via the OkHttp interceptor chain:

| HTTP Header | Interceptor | Purpose |
|-------------|-------------|---------|
| `App-Session-Id` | SessionHeaderInterceptor | Random session per app launch |
| `App-Session-Time-Elapsed` | SessionHeaderInterceptor | Anti-replay |
| `User-Session-Id` | SessionHeaderInterceptor | Per-user session (post-login) |
| `X-Install-Id` | InstallIdHeaderInterceptor | Stable install identifier |
| `X-Device-Model` | DeviceModelHeaderAppenderInterceptor | Device make/model |
| `X-Carrier` | DeviceCarrierHeaderAppenderInterceptor | Mobile carrier |
| `X-Sim-Info` | SimInformationHeaderAppenderInterceptor | SIM card details |
| `X-Persistent-Id` | PersistentIdHeaderInterceptor | Cross-install device ID |
| `Authorization` | TinderAuthHeaderInterceptor | Bearer token (post-auth) |

There are over 60 interceptors in total across modules like `api/`, `data/network/`, `platform/network/`, and `network/okhttp/cronet/`. The Cronet interceptors add HTTP/3 (QUIC) support with custom retry and authentication logic.

The protocol itself is straightforward:

1. Client sends `AuthRequest` (type + payload) to `POST /v2/auth`
2. AGS authenticates and returns an `AuthResult`
3. On `Authenticated`: save token, start `MainActivity`
4. On `Response(NextStep)`: push step to stack, launch next Activity
5. On `Error`: handle specific error code, retry or show ban screen

----

#### 8. What the Numbers Say

| Metric | Value |
|--------|-------|
| Decompiled Java files | 98,603 |
| Tinder own code files | 55,706 |
| Call graph nodes (methods) | 586,073 |
| Call graph edges (calls) | 1,491,572 |
| Auth module files | ~1,200 |
| AuthFlowStep states | 10 |
| AuthRequest variants | 27 |
| AuthStepV2 step types | 10 |
| OkHttp interceptors | 60+ |
| Activities | 90+ |
| FaceTec liveness files | 463 |
| Bugsnag + Sentry files | 914 |

The APK is heavily obfuscated with R8/ProGuard -- you can see single-letter package names (`a`, `a0`, `a1`, ..., `a9`, `aa`, `ac`...) that are clearly the obfuscator output. Despite this, garlic decompiled all 98,603 files to valid Java.

----

#### What I Learned

Building garlic as an MCP server changed how I approach APK analysis. Instead of a linear workflow (decompile, then grep, then manually trace), I can have an LLM drive the investigation: start with the manifest, pivot to the call graph to find entry points, query DuckDB for strings containing API URLs, then read the decompiled source for those specific classes. The whole loop stays in one conversation.

The DuckDB integration is the key enabler. A million-edge call graph is useless if you can't query it. Being able to write `SELECT method_raw FROM java_cg_nodes WHERE method_raw LIKE '%AuthGateway%'` and get results in milliseconds means you can explore the codebase at scale without knowing the file structure in advance.

The Tinder analysis was a good stress test -- 127MB APK, 98K files, heavy obfuscation, complex multi-step auth flow with 27 request types and 15+ error conditions. Garlic handled the decompilation and graph construction without issues.

The SQL interface made the auth flow analysis tractable. I started by searching the call graph for `AuthFlowStep` implementations to enumerate the state machine. Then I traced the `submitAuthRequestV2` calls to find the request/response types. The `string_nodes` table located all API endpoint URLs. Finally, reading the decompiled Java source confirmed the state transition logic, the error handling matrix, and the interceptor chain layout.

What would have been a full day of manual digging with `jadx` and `grep` took about 30 minutes of conversational probing through the MCP tools. The model drives the investigation; the tools provide the data.

The MCP server and garlic are both open source at [github.com/neocanable/garlic](https://github.com/neocanable/garlic) if you want to try it on your own APKs.
