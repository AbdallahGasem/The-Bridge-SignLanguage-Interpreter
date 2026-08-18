<div align="center">

<!-- Drop the logo here when you add it: <img src="docs/assets/logo.png" alt="The Bridge" width="150" /> -->

# The Bridge

### A two-way sign language interpreter for live video calls

**You sign, they read it. They speak, you read it. Same call, both directions, every model running on the phone in your hand.**

![Platform](https://img.shields.io/badge/platform-Android-3DDC84?logo=android&logoColor=white)
![Flutter](https://img.shields.io/badge/Flutter-02569B?logo=flutter&logoColor=white)
![TensorFlow Lite](https://img.shields.io/badge/TensorFlow%20Lite-FF6F00?logo=tensorflow&logoColor=white)
![WebRTC](https://img.shields.io/badge/WebRTC-peer--to--peer-333333?logo=webrtc&logoColor=white)
![Inference](https://img.shields.io/badge/inference-100%25%20on--device-006D6F)
![Status](https://img.shields.io/badge/status-working%20prototype-blue)

</div>

---

## Watch it work

| | |
|---|---|
| **[Demo.mp4](demo.mp4)** | The running system. Two Android devices, one signing, one speaking, captions appearing on both screens during a live call. |
| **[The_Bridge_Pitch.mp4](the_bridge_Pitch.mp4)** | The pitch for the idea, recorded at an earlier stage of the project. |

---

## The problem

Every day, a deaf person sits across from a doctor, a teller, a teacher, an employer, and neither person can speak to the other. There is no interpreter in the room, and there was never going to be one.

Interpreters are booked weeks ahead, billed by the hour, and physically present in exactly one place at a time. Most conversations in a life are not like that. They are unplanned, they are short, and they happen anyway — badly, through a family member, through a scrap of paper. Or not at all.

- **70M+** people belong to signing Deaf communities, using **300+** distinct sign languages. Only about 40% of national sign languages have any legal recognition.
- **California has one interpreter per 46 deaf people. Saudi Arabia has one per ~93,000.** That is not a shortage; it is a two-thousand-fold difference in caseload.
- **Egypt has an estimated 1.2M deaf and hard-of-hearing people** — and no maintained Egyptian Sign Language dictionary, no published grammar, and no continuous-signing corpus. To a machine-learning system, Egypt is a blank map.
- **Egypt's Law 10 of 2018, Article 19** already obliges hospitals, courts, police and broadcasters to provide sign-language access. The demand is not philanthropic. It is statutory, and it cannot currently be met.

A right that cannot be exercised is not a right. It is a promissory note. The Bridge is built to make that note payable — not by replacing interpreters, but by covering the vast majority of conversations no interpreter was ever going to attend.

---

## Why the existing options don't close it

| Option | Where it breaks |
|---|---|
| **Human interpreters** | The gold standard, and it must stay that way for diagnoses, courtrooms and contracts. But it is a scheduled, scarce, billed resource that cannot reach an unplanned two-minute exchange. |
| **Writing and lip-reading** | Lip-reading exposes only 30–40% of speech under optimal conditions. Writing assumes fluency in written Arabic, which for many deaf signers is a second language never fully acquired. |
| **Captions and ASR alone** | Solves exactly half the problem. A deaf person can now understand you; they still cannot answer you. A conversation is not a broadcast. |
| **Cloud AI and signing avatars** | Requires uploading video of the conversations people are least willing to upload, plus a live connection and a per-minute bill. Deaf peak bodies have jointly cautioned against avatars as substitutes for human signers. |
| **Academic sign recognition** | Hundreds of papers report 95%+ accuracy. Almost none ship, because those figures are measured on signers the model has already seen. |

---

## What The Bridge is

Not a concept, a mock-up, or a demo script. An application that runs on two ordinary Android phones, where two people who share no language can hold a conversation.

Two recognition streams run at once, on two devices:

- **The deaf user signs.** The camera feed goes to MediaPipe, which reduces each frame to a skeleton of hand landmarks. Those coordinates — not the video — feed a TensorFlow Lite classifier running natively on the phone. Recognised letters appear as text on the hearing user's screen.
- **The hearing user speaks.** The microphone feed goes to Vosk, an offline speech recogniser embedded in the app. The transcript appears as text on the deaf user's screen.

Neither stream leaves the device. Audio and video travel directly between the two phones over an encrypted peer-to-peer connection. A signaling server introduces the devices to one another, then plays no further part.

---

## Three refusals

The architecture is best described by what it declines to do. Each refusal costs something, which is how you can tell it is a commitment rather than marketing.

**The server never carries the conversation.** Media travels phone to phone over DTLS-SRTP. The signaling server relays the connection handshake and nothing else. It is architecturally incapable of observing a call, because no media is ever routed through it.

**The models never leave the phone.** Sign recognition and speech recognition both execute on the handset. No inference API, no GPU cluster, no per-minute cost, no upload. The industry-standard alternative — stream video to a cloud model — was rejected on the grounds that the conversations that most need interpreting are exactly the ones nobody will consent to upload.

**Only results cross the platform boundary.** Camera frames and audio buffers are captured, processed and discarded inside the native layer. What crosses into the application layer is a short string: a letter, a word, a transcript fragment. Moving media across that boundary is what makes hybrid mobile apps slow. Moving a few bytes is free.

The consequence is unusual and worth stating plainly: **there is nothing to leak.** The privacy guarantee is not a policy we promise to honour. It is a property of where the code runs.

---

## How it fits together

```
+------------------- Your phone (Flutter + native Kotlin) --------------------+
|                                                                             |
|   Camera --> MediaPipe landmarks --> TFLite classifier -----+                |
|                                                             +--> captions   |
|   Microphone --> Vosk offline ASR --------------------------+       |        |
|                                                                     v        |
|                        WebRTC peer connection ---- data channel -----+-------+--> Peer's phone
|                                                                             |
+-------------------------------+---------------------------------------------+
                                |  sign-in, contacts, call setup (never media)
              +-----------------+------------------+
              |                                    |
     Signaling server (NestJS)          Supabase (auth, profiles, friends)
```

Four components divide the work on the client:

- **MediaService** — sole owner of the camera and microphone. A single-owner rule that prevents the resource conflicts which otherwise plague apps doing capture and inference at once.
- **WebRTCService** — manages the peer connection.
- **SignalingService** — speaks Socket.IO to the backend.
- **CallController** — the state machine orchestrating the other three, so call state lives in exactly one place.

| Data | Where it lives | Leaves the device? |
|---|---|---|
| Camera frames | Device only | Only as encrypted call video, direct to the peer |
| Microphone audio | Device only | Only as encrypted call audio, direct to the peer |
| Sign and speech inference | On-device | Never |
| Recognised text and captions | Device + peer | To the peer only, over the encrypted data channel |
| Call setup (SDP / ICE) | Signaling server | Relayed once, then discarded. No media. |
| Accounts, profiles, friend graph | Supabase | Yes — the one cloud dependency |
| Conversation content | Nowhere | Not stored, not logged, not transmitted, even to us |

Identity lives in the cloud. The conversation does not.

---

## What ships today

The system is built around **language packs** — a pair of an offline speech model and a sign-recognition model. Adding a sign language means adding a pack, not rebuilding the application. That is a data problem, not an engineering one, and it is why the corpus below is the decisive next step rather than a new codebase.

| Capability | Status | Notes |
|---|---|---|
| 1:1 real-time video call | Working | Peer-to-peer WebRTC. Media never touches a server. |
| Sign to text: ASL fingerspelling | Working | 28 classes. On-device MediaPipe + TFLite. Runs live, during a call. |
| Speech to text: English | Working | Vosk, fully offline. Model bundled with the app. |
| Social graph: accounts, search, contacts, call initiation | Working | Supabase. An application, not a laboratory harness. |
| Speech to text: Arabic (MSA) | Integrating | Model works and runs offline. Side-loaded during development because of its size; on-demand download is a packaging task. |
| Live Mode: in-person, single device | In build | Point the phone at the person in front of you. Captions without a call — the mode that answers the shop counter and the hospital desk. |
| Sign to text: Arabic fingerspelling | In build | Same pipeline, different pack. |

---

## The recognition engines

Two engines, because fingerspelling is a held shape and a word sign is a movement. These are different problems, and one network forced to do both does neither well.

Both share a front-end: every camera frame goes to MediaPipe, which returns a normalised skeleton, and the frame itself is discarded. Everything downstream operates on geometry. That single decision buys four things at once — it collapses the payload from an image to a few hundred floats, removes nuisance variation in lighting and background, is **invariant to skin tone** in a way pixel-based vision models notoriously are not, and is privacy-preserving by construction.

**Engine B — what ships.** A compact multilayer perceptron, roughly 240 KB, classifying a single frame of hand landmarks into a letter. Behind it sit a stabiliser (confidence gate plus debounce, so a letter is emitted once rather than forty times as the hand is held) and an assembler that composes stabilised letters into words. This is the engine running in every live demonstration.

**Engine A — the research engine.** Whole word signs from motion. Each frame becomes a 225-dimensional vector — 33 pose landmarks and 21 per hand, in three dimensions, chest-anchored — and 100 frames form a sequence. Two bidirectional GRU layers read it forwards and backwards, so each moment is interpreted with knowledge of what follows it. Roughly 590,000 parameters, about 2.25 MB. **It is deliberately excluded from the shipping path**, because it does not yet meet the standard we would require before putting it in front of a deaf user.

---

## Accuracy, reported honestly

Sign-recognition papers routinely report above 95%. Those figures are, in the great majority of cases, inflated by **signer leakage**: the same person's hands appear on both sides of the split, so the model learns the signer rather than the sign. We measured it on our own system, and we publish both numbers.

| Configuration | Top-1 / Top-5 | What it actually means |
|---|---|---|
| Engine B, public benchmark, random split | **98.44%** | The number the literature would report. Inflated by signer leakage. |
| Engine B, held-out signers | **93.2%** | The figure a stranger picking up the phone should expect. |
| Engine A, signer-disjoint | **27.4% / 56.4%** | The honest figure for dynamic word recognition. |

A system that claims 98% and delivers far less in a hospital corridor is worse than useless. It is dangerous, and it discredits the field. We would rather report a smaller number that is true.

### The finding: the ceiling is data, not architecture

The instinct on seeing 27.4% is to blame the model. We tested that instinct. A regularisation and capacity sweep spanning a **three-fold range of model size moved top-1 accuracy barely at all** — a model three times larger did not learn appreciably more, and one three times smaller did not learn appreciably less.

That is not the signature of an under-powered architecture. It is the signature of a **data ceiling**: the training set does not contain enough distinct signers for any model of this family to generalise beyond them.

This is the most important result in the project, and it is a negative one. If the ceiling were architectural, the answer would be a better model, and better models are cheap. Because the ceiling is data, the answer is a corpus.

---

## Speech recognition

Powered by **Vosk**, running fully offline on-device. Real-time captions, no cloud dependency, no latency tax, no server cost.

The app runs its own native audio capture in parallel with the call rather than intercepting the WebRTC track — simpler, more robust, and it avoids fighting the framework at exactly the point where latency matters most. One consequence had to be handled explicitly: because capture is independent, muting the microphone has to pause the recogniser deliberately, or a muted user keeps transcribing.

The recogniser emits **partials** (continuous, provisional, driving the on-screen caption so text keeps pace with the speaker) and **finals** (committed at utterance end, sent to the peer). Almost all of the system's perceived responsiveness comes from that distinction.

Recognised text crosses to the other phone as a few hundred bytes on the data channel. It could have been re-synthesised as audio or rendered as a signing avatar. Text was chosen because it is robust on a weak connection, cheap, and honest — it does not impersonate a human voice or a human signer, and it makes no claim to be one.

| Language | Status |
|---|---|
| English | Live. Model bundled with the application. |
| Arabic (Modern Standard) | Integrating. Runs fully offline; packaging is a distribution task, not a research one. |
| Egyptian colloquial Arabic | Not characterised. A different register from MSA. Accuracy has not been measured, and we do not claim it. |

---

## What The Bridge does not do

A prototype that claims everything has been tested against nothing. The following are not built. They are stated here so that what is built can be trusted.

| Not built | Why, and what it would take |
|---|---|
| Continuous Egyptian Sign Language recognition | The single largest gap. Blocked on data, not architecture: no continuous-signing EGSL corpus exists. |
| Testing with deaf users | **No deaf user has yet evaluated this system.** This is the largest gap in the project, and the first item on the roadmap. |
| Runtime switching between sign models | Language packs exist as separate models; hot-swapping inside a live call is not implemented. |
| Sign-to-sign translation | Requires paired corpora that do not exist for any Arabic sign language. |
| TURN relay | Peer-to-peer currently relies on public STUN. Carrier-grade NAT on both ends can block a direct path and the call will fail to connect. A deployment task, scoped for the next release. |
| Automatic mode switching | The intended mechanism is a velocity gate on hand-keypoint motion. In the current build it is a user-facing toggle. |
| Medical or legal use | Accuracy has not been characterised for high-stakes settings, and we would oppose its use in them at the current state of the system. |
| iOS | Android only. The Flutter client and native inference layer are portable; iOS is a port, not a rebuild. |

---

## Augmentation, not replacement

The deaf community's own peak bodies have been explicit that automated systems must not be positioned as substitutes for qualified interpreters. We treat that as a boundary that determines what gets built, not a disclaimer appended at the end.

**A human interpreter is required** for medical diagnoses, court proceedings, legal contracts, job interviews, safeguarding. The Bridge is not appropriate there and does not claim to be.

**No interpreter was ever going to attend** the pharmacy counter, the bank teller, the taxi, the parent–teacher meeting, the government service desk, the delivery at the door. Unscheduled, short, constant, and currently conducted through a family member, a scrap of paper, or not at all. That is the entire territory The Bridge is built for.

Because it runs on a phone rather than through a booking desk, it does not compete for the interpreter capacity that high-stakes settings depend on. If anything, it protects that capacity by removing the routine load from it.

---

## What comes next: the corpus

The engineering is finished and running on real hardware. What is missing is a language, and languages are collected, not coded.

The critical path is a recording programme, built to a standard the existing Arabic datasets do not meet — the published Arabic alphabet corpora ship without participant identifiers, which makes signer-disjoint evaluation impossible on them and guarantees every accuracy figure derived from them is leaky.

| Corpus design | Specification |
|---|---|
| Signers | 100 distinct signers — enough to hold out a meaningful test population |
| Geographic spread | Multiple governorates. EGSL varies regionally by roughly 20%, so a single-city corpus encodes a single-city dialect. |
| Vocabulary | 800–1,000 signs, prioritised by everyday settings: pharmacy, service desk, transport, school |
| Participant identifiers | Recorded from the first session. The decision that makes honest evaluation possible, and the one existing datasets omitted. |
| Idle footage | Genuinely idle recordings for the null class, not the frames adjacent to signs |
| Governance | Deaf-led. Recorded with, and reviewed by, Egyptian deaf associations rather than about them. |
| Licence | Openly released |

There is currently no openly licensed, signer-identified Egyptian Sign Language corpus in existence. Building one does not merely unblock The Bridge — it unblocks every researcher, student and developer who has been unable to work on EGSL for the same reason we were.

**The application is the demonstration. The corpus is the public good.**

### Sequence

| | Today | 0–6 months | 6–18 months | 18 months + |
|---|---|---|---|---|
| **Language and data** | ASL fingerspelling live on-device; Engine A result published | EGSL corpus pilot, multi-governorate; Arabic fingerspelling pack | Full EGSL corpus released openly; continuous-sign engine retrained | Further Arab sign languages added as packs |
| **Product** | 1:1 calls, two-way captions, social graph | Live Mode (in-person); model hot-swap | iOS port; Egyptian colloquial speech recognition | Deployment kit for institutions |
| **Infrastructure** | P2P + STUN; signaling-only server | TURN relay; cross-carrier validation | On-demand language-pack delivery | — |
| **Community and validation** | **No deaf users have tested the system. Our largest gap.** | Partnership with Egyptian deaf associations; first user testing | Deaf-led corpus governance; field pilot in a clinic or service desk | Compliance pathway under Law 10/2018 |

---

## Team

Six final-year Information Systems students at the **Faculty of Computers and Artificial Intelligence, Cairo University**, supervised by **Prof. Ali Zidane El-Qutaany**. Defended July 2026, and continuing beyond graduation.

| | |
|---|---|
| **Abdallah Gasem El-Sayed** | Project lead. Subsystems integration. Engine A research and signer-disjoint evaluation methodology. Built the real-time communication module independently, and the native capture layer for audio and video. |
| **Ahmed Karam Gamal** | Speech recognition module and the end-to-end offline ASR pipeline. |
| **Enas Ragab Kamel** | Speech recognition module and the end-to-end offline ASR pipeline. |
| **George Sherif Nabil** | Engine B, the static fingerspelling classifier running in the live system. Social graph module. |
| **Mohamed Abdelhamid Wagdy** | Engine B contribution, and the post-processing pipeline for Engine A. |
| **Dina Gamal Kamal** | User and social graph modules. |

**Recognition:** Graded **A+ (193/200)**, first discussion 80/80. Submitted to **Prototypes for Humanity 2026**, Dubai.

---

## Repository status

This repository currently holds documentation and media. **The source code is not published here.**

Whether an open-source version is released — and under what licence — is a decision the team will make after the Prototypes for Humanity 2026 result. Until then the code remains private, and the material here is intended to let you evaluate the system without it.

If you want to see it running, start with [demo.mp4](demo.mp4).

---

## Contact

Interested as a collaborator, a partner, a deaf-led organisation, or a funder of the corpus? We would like to hear from you.

- **Abdallah Gasem El-Sayed** (project lead) — ag.ellsayed@gmail.com
- **Prof. Ali Zidane El-Qutaany** (supervisor) — a.zidane@fci-cu.edu.eg

---

## Licence

Copyright 2026 The Bridge team. All rights reserved. Licence to be determined.

<div align="center">

**Egypt wrote the promissory note in 2018. The engineering that would make it payable now exists, runs on an ordinary phone, and costs nothing per conversation. What stands between the two is a corpus of a language no one has yet been funded to record.**

</div>
