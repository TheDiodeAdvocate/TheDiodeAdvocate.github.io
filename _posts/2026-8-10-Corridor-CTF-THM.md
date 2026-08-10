---
layout: post
title: "Corridor CTF Notes"
date: 2026-08-10
---


# Corridor
platform: TryHackMeroom

## Overview
IDOR-style room: a corridor of clickable doors, each linking to a URL path
built from a 32-character string. Goal is to figure out how those path
strings are generated and use that to reach a door not directly given.

## Recon & Enumeration
- Observed the door URLs follow the pattern `http://<IP>/<32-char string>`
- Confirmed one sample string (`c4ca4238a0b923820dcc509a6f75849b`) as MD5:
  32 hex chars, valid hex encoding, MD5 always outputs 128-bit -> 32 hex chars
- That sample turned out to be a well-known reference value: MD5 of the
  single digit 1
- Working theory: door path = MD5 of some small/simple input (digit or
  short string), not a random token — worth testing digits/simple strings
  against the observed hashes

## Exploitation
- Confirmed the plaintext -> hash relationship (door path = MD5 of a small
  input, e.g. a digit/room number) and used it to reach the target door
  directly. Flag captured.
- Hash a known input directly to test guesses:
  ```
  echo -n "14" | md5sum
  ```
- Wordlist / brute-force approach with hashcat:
  ```
  hashcat -m 500 -a 0 hash.txt wordlist.txt
  hashcat -m 0 hash.txt /root/Desktop/Tools/wordlists/rockyou.txt --show
  ```

## Dead Ends & Mistakes
- Hit a `separator unmatched` error in hashcat — root cause was hash file
  formatting, not the attack itself. Fixes to check next time:
  - make sure hash.txt contains *only* the hash, nothing else
  - drop `--username` unless the file is actually `user:hash` formatted
  - watch for Windows line endings (`dos2unix hash.txt` / strip `\r`)
  - strip stray leading/trailing whitespace
- Initially wondered if the hash sequence could be predicted/extrapolated
  (e.g. guess MD5(14) from MD5(13) pattern) — confirmed this doesn't work.
  Hash functions are one-way and unpredictable by design; the only options
  are: compute it directly from a known plaintext, use a precomputed/rainbow
  table, or brute force with a wordlist.

## Flags / Answers
- Flag captured. Located at room 0 (not room 14, which is where I kept
  looking). Room description said "back where you came from" — room 0 makes
  sense in hindsight.

## Lessons Learned
- Fixed-length hex strings in a URL are a strong tell for a hash — worth
  identifying the hash type early (length + charset) before doing anything else.
- For IDOR rooms built on hashes: the win condition is usually "find the
  plaintext -> hash relationship," not "crack this one hash and stop."
- hashcat "separator unmatched" is almost always a hash-file formatting
  issue, not a wrong command/mode.
- Re-reading the room's task description carefully is worth doing before
  brute-forcing anything — it usually names the vuln class directly (IDOR
  here) and hints at the encoding involved.
- Take room-description hints literally: "back where you came from" pointed
  at room 0, but I fixated on room 14 instead of testing the obvious low
  number first.
