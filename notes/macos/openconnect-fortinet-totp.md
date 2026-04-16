# Automated Fortinet SSL-VPN password and TOTP via macOS Keychain

FortiClient is [a pain](https://community.fortinet.com/t5/Support-Forum/FortiClient-VPN-disconnect-on-screen-lock-macOS/m-p/423540) on macOS.


## `openconnect` to te rescue

Did not work for me:

```bash
printf '%s\n' "$PASSWORD" |  sudo openconnect ... --token-mode=totp --token-secret='@./totpsecretfilepath' --passwd-on-stdin
# fgets (stdin): Inappropriate ioctl for device
```

The following is secure-ish, though mind that it defeats the purpose of the two/multi factor authentication (2FA/MFA) purpose of the OTP when storing them in the same place (macOS Keychain). But at least we are not saving the credentials in plain text files or passing them directly as arguments...

Fortinet supports appending the TOTP to the password which makes the authentication flow simpler.

```bash
VPN_USERNAME="yourusername"
VPN_GATEWAY="access.gateway.example.com"
# Added with: `security add-generic-password -a "$VPN_USERNAME" -s "$KEYCHAIN_PASSWORD" -w`
KEYCHAIN_PASSWORD="some_unique_id" # CHANGE ME
KEYCHAIN_TOTP="some_unique_id_totp" # CHANGE ME
CERT="pin-sha256:EXAMPLE" # CHANGE ME, optional
# It will ask for your macOS password to access them.
SECRET="$(security find-generic-password -a $VPN_USERNAME -s $KEYCHAIN_TOTP -w)"
PASSWORD="$(security find-generic-password -a $VPN_USERNAME -s $KEYCHAIN_PASSWORD -w)"

# Pass credentials always on stdin! (processes listing can leaks the arguments)
# `printf` is a builtin, no external process is spawned (hence cannot be leaked).
# `printf` if generally safer than `echo`.
OTP="$(printf '%s' "$SECRET" | oathtool --totp -b -)"
PW="${PASSWORD}${OTP}"
unset SECRET PASSWORD OTP

printf '%s' "$PW" | sudo openconnect \
  --protocol=fortinet \
  "$VPN_GATEWAY" \
  -u "$VPN_USERNAME" \
  --servercert "$CERT" \
  --non-inter \
  --passwd-on-stdin
```