---
title: "Attackers are repackaging, defenders are unaware. Why PLAY_RECOGNIZED is not enough."
date: 2026-10-22
description: "Repackaging is an attack class easy to scale. Google Play Integrity's PLAY_RECOGNIZED verdict proves where an Android app came from, not that its running code is unmodified, and we show why."
tldr: "We repackaged an app, ran it on Android 16, left the original signing certificate in place, and Google Play Integrity returned PLAY_RECOGNIZED. Play Integrity's not enough anymore."
---
Apps are easy to repackage.
Someone pulls your app <i>(adb pull /path/YOURAPP.apk)</i> and changes its behavior: the camera is injected, the device looks clean, ads are stripped or paywalls are removed. That is repackaging.

<u><b><i>Why do we care?</i></b></u> ~ Your server-side decisions are based on device-based signals. If every isRooted, isHooked, isFrench (😨) check returns "false" you cannot tell the difference between good traffic and someone harming your app.
This is particularly dangerous for <u>Fintech</u>, <u>Crypto</u>, <u>Web3</u> and <u>gambling apps</u> that have sensitive code to protect.
<u><b><i>Why is it overlooked?</i></b></u> ~ Because security teams are academically trained to tackle the most <i>powerful</i> attacks, not the most <i>scalable and profitable</i>.
And while your red team plays capture-the-flag on your paycheck trying to patch exotic exploits, a carder (someone who cashes out stolen payment cards) in his basement with a $20 ChatGPT subscription is three prompts away from fooling your KYC verifications.

## Google certifies code it never once looked at

On Android, Google Play Integrity is supposed to catch this through PLAY_RECOGNIZED. We traced the whole check on a live device and the process is straightforward:

1. The app calls the Play Integrity API.
2. <i>`com.google.android.finsky.integrityservice.IntegrityService`</i> takes the request and calls DroidGuard. 
3. <i>`com.google.android.gms.droidguard.DroidGuardService`</i> collects the package name, version code, install source, device id, SIM value and the request nonce into a hardware-backed attestation.
4. Google's servers decide the verdict and sign it server-side.
5. For `PLAY_RECOGNIZED` Google compares just two things: your app's package name and its signing certificate.
6. <u><b>If the certificate does not match</b></u>, you get <b>`UNRECOGNIZED_VERSION`</b>.
<u><b>If both match</b></u>, you get <b>`PLAY_RECOGNIZED`</b>. 
<i>("the app and certificate match the versions distributed by Google Play." https://developer.android.com/google/play/integrity/verdicts)</i>

<b>Google only checks the certificate on record</b>, fixed the moment the app was installed, regardless of runtime state. 
We proved it on a rooted Android 16 phone <i>(Google Play Services 26.34.67)</i> by repackaging <i>gr.nikolasspyr.integritycheck</i>: changed the code, re-signed it with our own key. Same package name, same version, different signing key:

```
genuine build      signer SHA-256  17952b5c...   CN=Android, O=Google Inc.
repackaged build   signer SHA-256  2839c45a...   CN=resign
```

Android will not let you update the installed genuine app with the repackaged one. It checks the signer and refuses:

```
INSTALL_FAILED_UPDATE_INCOMPATIBLE: Existing package signatures do not match newer version; ignoring!
```

→ <b><u>This is the whole defense Play Integrity offers</b></u>. ←
With root we left the genuine install record untouched (so the system still had the genuine certificate on file) and overwrote the actual code file on disk with our repackaged build:

```
# with root: swap the code, leave the install record alone
su -c "cat repackaged.apk > /data/app/~~.../gr.nikolasspyr.integritycheck-.../base.apk"
```

Then we asked Play Integrity for a verdict. For code that was ours, not Google's, it returned:

```json
{
  "appIntegrity": {
!!     "appRecognitionVerdict": "PLAY_RECOGNIZED",
    "packageName": "gr.nikolasspyr.integritycheck",
    "certificateSha256Digest": ["17952b5c..."],
    "versionCode": "22"
  },
!!   "accountDetails": { "appLicensingVerdict": "LICENSED" },
!!   "environmentDetails": { "playProtectVerdict": "NO_ISSUES" }
}
```

<b>The attacker's code was running, and every check still reported genuine.</b>

We also made sure the changed code actually ran, and was not a leftover compiled copy. We deleted the phone's pre-built copy of the app, so Android had to load our modified code from the swapped file, and a small marker we added to that code wrote to the log from the very same app process that then received the `PLAY_RECOGNIZED` verdict.

The code on that phone was the attacker's, signed `2839c45a...`. Google reported the genuine certificate, `17952b5c...`: `PLAY_RECOGNIZED`, `LICENSED`, Play Protect `NO_ISSUES`. The verdict reads the certificate cached at install; it never re-hashes the running code. A repackaged app carries a spotless, genuine verdict.

## For a high-value app, three holes make it worse

If the app lives in fintech, crypto, web3, social, or gambling, the verdict leaves three openings:

1. <u><b>The Play Integrity check is proven falsifiable.</u></b> A genuinely-signed app, hooked at runtime, runs changed behavior while the certificate on record stays the same. The verdict still says `PLAY_RECOGNIZED`.
2. <u><b>Off Google's reach, you get no usable verdict.</u></b> On a de-Googled device (one running without Google services, such as many China-market phones or a custom version of Android called a custom ROM), the Play Integrity check cannot run at all, so there is no verdict to act on. That forces a lose-lose choice on the defender: block everyone and lose real traffic, or allow everyone and let the attacker in.
3. <u><b>You get rate-limits and latency.</u></b> Quotas and latency forces the defender to check only on sensitive actions, leaving gaps the attacker can abuse.

On-device anti-repackaging alone has never held: a 2022 study [bypassed every repackage-proofing scheme it tested with a single attack toolkit](https://ink.library.smu.edu.sg/sol_research/3494), and the attacker's tools have only sharpened since. That is why the check that counts has to live off the device.

## If Play Integrity is unreliable, how do I defend my app?

<b>You <u>can</u> protect your app</b>, and we do that by making it so hard that it becomes economically unviable for them, so <u><b>the offenders will damage your competitors instead.</u></b>
<b><u>But how?</b></u> Repackaging alone without runtime hook and root detection is like locking your door but leaving the windows open. There's not many free tools online that can do both free, but one of them is BlackHex. BlackHex reads the code at runtime: it re-hashes what is actually loaded and compares it to the build shipped, covering the blind-spot that Play Integrity doesn't responsibly defend. The signer gets read two ways, through a raw native call and through the Android API, so a hook that lies to one is caught by the other. 

Indie developers and smaller apps don't need any high-ticket SDKs and are well served by the free tools already online ([BlackHex free tier](https://github.com/areyougarou/BlackHex), or [freeRASP](https://github.com/talsec/Free-RASP-Community)). Paid anti-fraud SDKs are suggested <b><u>only</b></u> to high-value targets (web3, crypto, Fintech, ...) where reducing fraud without slowing the app or destroying the conversion rate is one of the best and most measurable investments that can be made.

## The short version

Repackaging scales because the attacker does the work once and the copy runs on every device. For fintech, crypto, web3, and gambling apps, that copy can strip a paywall, fake an identity check, or hide that the phone is rooted. Google Play Integrity tries to stop repackaging but it's easy to spoof and we prove it in this article. Anti-fraud SDKs are mandatory for sensitive apps, BlackHex is a rare example of a good free-tier anti-fraud SDK. 