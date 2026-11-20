# ntfy Push Notifications

Each enrollment runs its own private [ntfy](https://ntfy.sh) server. UnitySVC Labs packages the official `binwiederhier/ntfy` image and the platform runs one isolated instance per enrollment.

1. Enroll in **base**.
2. The enrollment form shows a generated **Access token**; keep it or replace it with your own (the form `tk_` followed by 29 letters, digits, `-` or `_`), then save it. It is stored with the enrollment (view and copy it from the enrollment's instance details) and given to ntfy at start-up. It is an application credential, not your UnitySVC API key. Changing it later restarts the instance with the new token.
3. Wait for the instance to become ready. Its address is `https://<enrollment-code>.<apps-domain>` (HTTPS, port 443); the instance details show it.
4. Publish with the token as a bearer credential:

   ```bash
   curl -H "Authorization: Bearer tk_..." \
        -H "Title: Hello" -H "Priority: high" -H "Tags: wave" \
        -d "Backup finished" https://<instance>/<topic>
   ```

   Any topic name works; pick one that is hard to guess if you share the token widely.
5. Subscribe from the ntfy app (iOS, Android, desktop, web) by adding your instance URL as the server and the token as the credential, or stream with `curl -H "Authorization: Bearer tk_..." https://<instance>/<topic>/json`.

All ntfy publishing features work: priorities, tags, click actions, scheduled delivery, and attachments, with the limits below. The [ntfy documentation](https://docs.ntfy.sh/) describes them.

## Limits and storage

- Messages are limited to 4096 bytes and kept for 12 hours.
- Storage is temporary. Stopping, replacing or deleting the instance loses cached messages and any subscription state held on the server. No persistent volume is promised.
- Every request is authenticated; unauthenticated requests are denied. Keep the token private.
- iOS delivery: a self-hosted ntfy server has no direct path to Apple's push service, so iOS notifications can be delayed until the app is opened. Android delivers via a persistent connection.

## Older gateway endpoint

This service no longer routes through the UnitySVC API gateway, and your UnitySVC API key no longer authenticates ntfy requests. Use the instance URL and the access token instead.
